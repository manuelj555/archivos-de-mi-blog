---
layout: post
title: Formularios Dinamicos en Symfony
---

Este artículo intenta servir de guía para el entendimiento y la realización de formularios que tienen partes dinamicas (selects dependientes, campos adicionales dependientes, etc).

En symfony los formularios generalmente son clases php que extienden de una clase base llamada [AbstractType](http://symfony.com/doc/current/book/forms.html#creating-form-classes) y definen un método `buildForm` donde añadimos todos los campos que nuestro formulario tendrá.

El problema viene cuando algunos de nuestros campos dependen de valores establecidos por el usuario en otros campos, por ejemplo al seleccionar un pais en un select, el campo **estado** debería solo contener los estados de dicho pais; Otro ejemplo típico, es cuando marcamos alguna opción especifica o checkbox en un formulario, y aparecen campos adicionales. Todos estos casos necesitan de un trabajo adicional tanto en el servidor como en el cliente para su correcto funcionamiento.
 
## Creando un Nuevo Usuario
 
 Veamos un ejemplo básico de como implementar un campo **estado** dinamico, que depende de la selección que haga el usuario en otro campo **pais**:
 
{% highlight php linenos %}
<?php
# src/AppBundle/Form/Type/UserType.php

namespace AppBundle\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolverInterface;

class UserType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('country', 'entity', array('class' => 'AppBundle\Entity\Country'))
            ->add('state', 'entity', array('class' => 'AppBundle\Entity\State'));
        
        // Añadimos un EventListener que actualizará el campo state
        // para que sus opciones correspondan
        // con el pais seleccionado por el usuario
        $builder->addEventSubscriber(new AddStateFieldSubscriber());
    }

    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults(array(
            'data_class' => 'AppBundle\Entity\User',
        ));
    }

    public function getName()
    {
        return 'user';
    }

}
{% endhighlight %}

Symfony, mediante la implementación de eventos del formulario, permite que el desarrollador acceda a los datos enviados por el usuario al hacer submit del form, y así poder actualizar campos, actualizar la data, o validar dicha data.

En este caso, la clase `AddStateFieldSubscriber` será la encargada de actulizar el campo `state` en base al pais seleccionado por el usuario:

{% highlight php linenos %}
<?php

namespace AppBundle\Form\Listener;

use Symfony\Component\Form\Form;
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Doctrine\ORM\EntityRepository;

class AddStateFieldSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents()
    {
        return array(
            FormEvents::PRE_SUBMIT => 'preSubmit',
        );
    }

    /**
     * Cuando el usuario llene los datos del formulario y haga el envío del mismo,
     * este método será ejecutado.
     */
    public function preSubmit(FormEvent $event)
    {
        $data = $event->getData();
        //data es un arreglo con los valores establecidos por el usuario en el form.

        //como $data contiene el pais seleccionado por el usuario al enviar el formulario,
        // usamos el valor de la posicion $data['country'] para filtrar el sql de los estados
        $this->addField($event->getForm(), $data['country']);
    }

    protected function addField(Form $form, $country)
    {
        // actualizamos el campo state, pasandole el country a la opción
        // query_builder, para que el dql tome en cuenta el pais
        // y filtre la consulta por su valor.
        $form->add('state', 'entity', array(
            'class' => 'AppBundle\Entity\State',
            'query_builder' => function(EntityRepository $er) use ($country){
                return $er->createQueryBuilder('state')
                    ->where('state.country = :country')
                    ->setParameter('country', $country);
            }
        ));
    }
}

{% endhighlight %}

Ahora, cuando el usuario envie el formulario, el método `AddStateFieldSubscriber::preSubmit` será invocado, y el campo `state` se actualizará con los estados del pais seleccionado por el usuario.

#### Actualizando los Estados en el Navegador

Hasta ahora hemos logrado que en el servidor, cuando se envie el formulario el select de los estados actualize sus opciones en base al pais seleccionado, pero aun necesitamos que cuando el usuario seleccione un pais el campo `state` se actualize y muestre los estados de dicho pais.

Para lograrlo vamos hacer uso de javascript y jquery siguiendo el ejemplo de la [documentación oficial](http://symfony.com/doc/current/cookbook/form/dynamic_form_modification.html#dynamic-generation-for-submitted-forms), creando un evento `onChange` para el campo `country` donde vamos a ejecutar un llamado ajax al servidor, que nos permita obtener los estados actualizados en base al pais seleccionado.

Lo primero que haremos será crear el formulario en el controlador y mostrarlo en la vista:
 
{% highlight php linenos %}
<?php
# src/AppBundle/Controller

namespace AppBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use AppBundle\Entity\User;
use AppBundle\Form\Type\UserType;
use Symfony\Component\HttpFoundation\Request;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;

class RegisterController extends Controller
{

    /**
     * @Route("/new", name="user_new")
     */
    public function newAction(Request $request)
    {
        $form = $this->createForm(new UserType(),$user = new User());
        $form->handleRequest($request);
        
        if($form->isSubmitted() and $form->isValid()){
            $em = $this->getDoctrine()->getManager();
            
            $em->persist($user);
            $em->flush();
            
            return $this->redirectToAction('alguna acción de la app');
        }
        
        return $this->render('user/new.html.twig', array(
            'form' => $form->createView(),
         ));
    }

}
{% endhighlight %}

Ahora en la vista twig, mostraremos el formulario y añadiremos el javascript necesario para que el campo `state` se actualize cuando el usuario seleccione o cambie de pais:

{% highlight js linenos %}
{% raw %}
{# app/Resources/views/user/new.html.twig #}

{{ form_start(form) }}
    {{ form_row(form.country) }}    {# <select id="user_country" ... #}
    {{ form_row(form.state) }} {# <select id="user_state" ... #}
    {# ... #}
{{ form_end(form) }}

<script type="javascript">
    var $country = $('#user_country'); 
    //tambien podemos hacer $('#{{ form.country.vars.id }}') para obtener el id
    
    var $form = $country.closest('form');

    // cada vez que el usuario cambie el pais en el select
    $country.on('change', function() {

        // creamos la data, solo con el campo del pais,
        // ya que es el dato relevante en este caso.
        var data = $country.serialize();

        // Hacemos un envío del formulario, lo que ejecutará el evento preSubmit
        // del listener AddStateFieldSubscriber,
        // y actualizará el campo state, con los estados del pais seleccionado.

        $.ajax({
            url : $form.attr('action'),
            type: $form.attr('method'),
            data : data,
            success: function(html) {

                // la variable html representa toda la página junto con el select de estados.
                // el cual tomamos y colocamos para reemplazar el select actual.

                $('#user_state').replaceWith($(html).find('#user_state'));
            }
        });
    });
</script>

{% endraw %}
{% endhighlight %}

Este código javascript lo que hace es enviar una petición ajax a la misma página que procesa el formulario, para forzar la ejecución del evento `PRE_SUBMIT` del formulario y hacer que el listener `AddStateFieldSubscriber` actualize el campo `state` en base al pais enviado. Por ultimo, tomamos de la respuesta html el campo `#user_state` y lo colocamos en reemplazo del campo original en la página.

Es importante destacar que para poder implementar esta reutilización de la acción `newAction` para solo actualizar el campo `state`, debemos estar completamente seguros de que el ajax, solo está enviando los datos minimos necesarios, haciendo que el formulario seá invalido conscientemente, ya que de otra forma, si enviamos toda la data, o con la data minima enviada el formulario pasa las validaciones, se va hacer el insert del usuario en la base de datos.

En este caso, una de las cosas por las que podemos saber que el formulario no pasa el proceso de validación, es porque no estamos enviando el token de seguridad de los formularios symfony.

Si por otro lado queremos estar más seguros de que no se vaya hacer un insert indeseado en la base de datos, podemos crearnos una nueva acción que solo cree y muestre el formulario (que acepte además solo peticiones ajax), pero que nunca haga el persist del objeto.

{% highlight php linenos %}
<?php
# src/AppBundle/Controller

namespace AppBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use AppBundle\Entity\User;
use AppBundle\Form\Type\UserType;
use Symfony\Component\HttpFoundation\Request;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;

class RegisterController extends Controller
{
    /**
     * @Route("/new", name="user_new")
     */
    public function newAction(Request $request)
    {
        ...
    }
    
    /**
     * @Route(
     *      "/ajax-form", 
     *      name="user_ajax_form",
     *      conditions="request.isXmlHttpRequest()"
     * )
     */
    public function ajaxFormAction(Request $request)
    {
        $form = $this->createForm(new UserType(),$user = new User());
        $form->handleRequest($request);
        
        return $this->render('user/new.html.twig', array(
            'form' => $form->createView(),
         ));
    }

}
{% endhighlight %}

Ahora con solo hacer que en el javascript, la url del método $.ajax de jquery apunte a la ruta `user_ajax_form` tendremos la seguridad de que nunca se va hacer algún persist indeseado.

{% highlight javascript linenos %}
{% raw %}
// Antes:
$.ajax({
    url : $form.attr('action'),
    
// Ahora:
$.ajax({
    url : "{{ path('user_ajax_form') }}",
{% endraw %}
{% endhighlight %}

## Editando un Usuario Existente

El ejemplo anterior nos permitió actualizar los estados en base a un pais seleccionado por el usuario desde el navegador. Pero ¿que pasa cuando vamos a editar un usuario previamente guardado, donde este ya tiene un estado seleccionado con anterioridad?

En ese caso, deberían aparecer listados los estados del pais previamente seleccionado al momento de mostrar el formulario de edición por primera vez.

Vamos a mostrar dos situaciones y como resolver este problema en cada una de ellas.

#### Primer caso
**La entidad `User` posee tanto el atributo `state` como el atributo `country`**:

Como ambos atributos se encuentran en la entidad User, tanto el campo `state` como el campo `country` estarán asociados al formulario y cuando se cree el form de edición, el pais aparecerá seleccionado en base a los valores persistidos en la base de datos.

Entonces solo debemos hacer que el campo `state` tome en cuenta el pais que contiene la entidad `User` y en base a dicho pais, cree las opciones del select de los estados. Para lograrlo, vamos a modificar la clase  `AddStateFieldSubscriber` para que escuche el evento `pre_set_data` de los formularios de symfony, ya que este evento se ejecuta justo antes de establecer los valores de cada campo del formulario en base a los datos almacenados en el objeto de la clase `User`.

{% highlight php linenos %}
<?php

namespace AppBundle\Form\Listener;

use Symfony\Component\Form\Form;
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Doctrine\ORM\EntityRepository;

class AddStateFieldSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents()
    {
        return array(
            FormEvents::PRE_SET_DATA => 'preSetData', //nuevo evento escuchado
            FormEvents::PRE_SUBMIT => 'preSubmit',
        );
    }
    
    /**
     * Este evento se ejecuta al momento de crear el formulario 
     * o al llamar al método $form->setData($user),
     * y nos sirve para obtener datos inicales del objeto asociado al form.
     * Ya que por ejemplo si el objeto viene de la base de datos y contiene
     * ya un pais establecido, lo ideal es que el campo state se carge inicalmente con
     * los estados de dicho pais.
     */
    public function preSetData(FormEvent $event)
    {
        $user = $event->getData(); //data es un objeto AppBundle\Entity\User

        // Pasamos siempre el country así sea null
        // para que cuando sea un usuario nuevo, el listado de estados esté
        // vacio inicialmente, y solo se llene de items, cuando se ejecute el 
        // ajax que obtiene los estados del pais seleccionado por el usuario.

        $country = ($user and $user->getCountry()) ? $user->getCountry() : null; // Importante los parentesis al usar "and".
        
        // Es importante siempre verificar que el valor devuelto por $event->getData()
        // (que en este caso es $user) no sea null, porque no es obligatorio que al crear
        // el formulario, se le pase una instancia de User,
        // y si no se le pasa, User será nulo.

        $this->addField($event->getForm(),  $country);
    }

    /**
     * Cuando el usuario llene los datos del formulario y haga el envío del mismo,
     * este método será ejecutado.
     */
    public function preSubmit(FormEvent $event)
    {
        $data = $event->getData();
        //data es un arreglo con los valores establecidos por el usuario en el form.

        //como $data contiene el pais seleccionado por el usuario al enviar el formulario,
        // usamos el valor de la posicion $data['country'] para filtrar el sql de los estados
        $this->addField($event->getForm(), $data['country']);
    }

    protected function addField(Form $form, $country)
    {
        // actualizamos el campo state, pasandole el country a la opción
        // query_builder, para que el dql tome en cuenta el pais
        // y filtre la consulta por su valor.
        $form->add('state', 'entity', array(
            'class' => 'AppBundle\Entity\State',
            'query_builder' => function(EntityRepository $er) use ($country){
                return $er->createQueryBuilder('state')
                    ->where('state.country = :country')
                    ->setParameter('country', $country);
            }
        ));
    }
}
{% endhighlight %}

El nuevo método `AddStateFieldSubscriber::preSetData` ahora actualizará siempre el campo `state` al crearse el formulario, por lo que ya no es necesario añadir este campo en el método `buildForm` del `UserType`:

{% highlight php linenos %}
<?php
# src/AppBundle/Form/Type/UserType.php

public function buildForm(FormBuilderInterface $builder, array $options)
{
    $builder
        ->add('country', 'entity', array('class' => 'AppBundle\Entity\Country'));
        // ya no hace falta agregar el campo state acá, ya que el Listener
        // lo va a reemplazar siempre al crear el formulario.
        //->add('state', 'entity', array('class' => 'AppBundle\Entity\State'));
    
    $builder->addEventSubscriber(new AddStateFieldSubscriber());
}
{% endhighlight %}

#### Segundo caso
**La entidad `User` solo posee el atributo `state` y el country se obtiene desde dicho atributo**:

Algunas veces para ahorrar espacio en la base de datos, optamos por no colocar el atributo del pais en la entidad `User`, ya que podemos llegar a el por medio de la propiedad `state` así:

{% highlight php linenos %}
<?php

$user->getState()->getCountry();
{% endhighlight %}

El problema de esta implementación es que cuando vamos a editar un registro, como el campo `country` no está mapeado con la entidad, al mostrar el formulario no va a aparecer seleccionado el pais previamente escogido por el usuario.

Para resolver este punto nos vamos a valer nuevamente de los eventos de formulario, y vamos a utilizar el evento `pre_set_data` para obtener el pais por medio del estado previamente guardado en la base de datos. Para efectos del ejemplo vamos a crear un listener dentro del propio formulario, pero si quieremos podemos crearnos una clase listener como la que se creó para manejar el campo `state`:

{% highlight php linenos %}
<?php
# src/AppBundle/Form/Type/UserType.php

public function buildForm(FormBuilderInterface $builder, array $options)
{
    // como vamos a crear el campo country en el evento pre_set_data
    // no hace falta añadirlo al builder.

    $builder->addEventSubscriber(new AddStateFieldSubscriber());
    
    // recordar importar las clases FormEvents y FormEvent
    $builder->addEventListener(FormEvents::PRE_SET_DATA, function(FormEvent $event){
        $user = $event->getData();
        $form = $event->getForm();
        
        if($user and $user->getState()){
            // obtenemos el country por medio del objeto state:
            $country = $user->getState()->getCountry();
        }else{
            $country = null;
        }
        
        $form->add('country', 'entity', array(
            'class' => 'AppBundle\Entity\Country',
            'mapped' => false, // importante indicar que el campo no está mapeado
            'data' => $country, //establecemos el valor inicial del campo.
        ));
    });
}
{% endhighlight %}

Ahora cuando el formulario sea creado, el campo country tendrá seleccionado el pais al que pertenece el estado que el usuario escogió. 

Es importante resaltar que el campo `country` ahora no está mapeado (`mapped => false`) ya que si no lo indicamos, el formulario intentará leer el valor del country desde la clase `User` y el framework lanzará una excepción indicando que no encontró un método en dicha clase para obtener el country.

## Listeners con Dependencias

Aveces se dan casos donde los listeners de un formulario dependen de servicios externos para poder realizar ciertas tareas (el EntityManager, el Token de la Sesión o el SecurityContext por ejemplo), y pasar esas dependencias al formulario para luego hacerlas llegar a los listeners puede ser muy complejo y hasta incorrecto.

En estos casos la solución más idonea es registrar el listener como un servicio e inyectarle las dependencias al listener. 

#### Veamos un Ejemplo:

Tenemos el caso del campo `state` que depende del campo `country`, vamos a añadir una condición que va a permitir modificar estos campos en la edición, solo si el usuario logueado es un Super Administrador.
  
Lo primero será modificar el listener `AddStateFieldSubscriber` para que haga uso de la clase `AuthorizationChecker` (añadida en Symfony 2.6) que será la encargada de verificar que el usuario logueado sea un administrador.

{% highlight php linenos %}
<?php

namespace AppBundle\Form\Listener;

use Symfony\Component\Form\Form;
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Doctrine\ORM\EntityRepository;

class AddStateFieldSubscriber implements EventSubscriberInterface
{
    protected $authorizationChecker;
    
    public function __construct(AuthorizationCheckerInterface $authorizationChecker)
    {
        $this->authorizationChecker = $authorizationChecker;
    }

    public static function getSubscribedEvents() { ... }
    
    public function preSetData(FormEvent $event) { ... }

    public function preSubmit(FormEvent $event) { ... }

    protected function addField(Form $form, $country)
    {
        //si se está en edición y el usuario no es super admin entonces el campo va deshabilitado
        $isEdit = $form->getData() instanceOf User and $form->getData()->getId();
        $disabled = $isEdit and !$this->authorizationChecker->isGranted('ROLE_SUPER_ADMIN');
    
        // actualizamos el campo state, pasandole el country a la opción
        // query_builder, para que el dql tome en cuenta el pais
        // y filtre la consulta por su valor.
        $form->add('state', 'entity', array(
            'class' => 'AppBundle\Entity\State',
            'disabled' => $disabled,
            'query_builder' => function(EntityRepository $er) use ($country){
                return $er->createQueryBuilder('state')
                    ->where('state.country = :country')
                    ->setParameter('country', $country);
            }
        ));
    }
}
{% endhighlight %}

Lo que hemos hecho es añadir una dependencia a `AuthorizationCheckerInterface` para poder verificar en el método `addField` si el usuario es un SuperAdmin. Además para saber si se está editando el registro, hacemos uso del método `$form->getData()` que nos debe devolver la instancia del objeto `User`, y por medio del valor del id sabemos si es un usuario nuevo o un usuario cargado de la base de datos.

Ahora creamos el listener del campo `country`, donde basicamente vamos hacer el mismo trabajo:

{% highlight php linenos %}
<?php

namespace AppBundle\Form\Listener;

use Symfony\Component\Form\Form;
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Doctrine\ORM\EntityRepository;

class AddCountryFieldSubscriber implements EventSubscriberInterface
{
    protected $authorizationChecker;
    
    public function __construct(AuthorizationCheckerInterface $authorizationChecker)
    {
        $this->authorizationChecker = $authorizationChecker;
    }

    public static function getSubscribedEvents()
    {
        return array(
            FormEvents::PRE_SET_DATA => 'preSetData',
        );
    }
    
    public function preSetData(FormEvent $event)
    {
        $user = $event->getData();
        $form = $event->getForm();
        
        if($user and $user->getState()){
            // obtenemos el country por medio del objeto state:
            $country = $user->getState()->getCountry();
            $isEdit = true;
        }else{
            $country = null;
            $isEdit = false;
        }
        
        //si se está en edición y el usuario no es super admin entonces el campo va deshabilitado
        $disabled = $isEdit and !$this->authorizationChecker->isGranted('ROLE_SUPER_ADMIN');
        
        $form->add('country', 'entity', array(
            'class' => 'AppBundle\Entity\Country',
            'mapped' => false, // importante indicar que el campo no está mapeado
            'data' => $country, //establecemos el valor inicial del campo.
        ));
    }
}
{% endhighlight %}

Con los listeners ya creados y actualizados, procedemos a registrarlos como servicios en la aplicación:

{% highlight yaml linenos %}
# app/config/services.yml
services:
    form.listener.add_state_field:
        class: AppBundle\Form\Listener\AddStateFieldSubscriber
        arguments: [@security.authorization_checker]
        
    form.listener.add_country_field:
        class: AppBundle\Form\Listener\AddCountryFieldSubscriber
        arguments: [@security.authorization_checker]
{% endhighlight %}

El servicio que representa una implementación de la interfaz `AuthorizationCheckerInterface` en Symfony es `security.authorization_checker`, y es este servicio el que inyectamos en el constructor de los listeners `AddCountryFieldSubscriber` y `AddStateFieldSubscriber`

Luego de registrar los listeners debemos actualizar el formulario para que haga uso de los servicios, y no cree las instancias directamente:

{% highlight php linenos %}
<?php
# src/AppBundle/Form/Type/UserType.php

namespace AppBundle\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolverInterface;

class UserType extends AbstractType
{
    protected $addStateFieldSubscriber;
    protected $addCountryFieldSubscriber;
    
    public function __construct($addStateFieldSubscriber, $addCountryFieldSubscriber)
    {
        $this->addStateFieldSubscriber = $addStateFieldSubscriber;
        $this->addCountryFieldSubscriber = $addCountryFieldSubscriber;
    }

    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder->addEventSubscriber($this->addStateFieldSubscriber);
        $builder->addEventSubscriber($this->addCountryFieldSubscriber);
    }

    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults(array(
            'data_class' => 'AppBundle\Entity\User',
        ));
    }

    public function getName()
    {
        return 'user';
    }

}
{% endhighlight %}

Y poy último en el controlador, pasamos los listener al formulario:

{% highlight php linenos %}
<?php
# src/AppBundle/Controller

namespace AppBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use AppBundle\Entity\User;
use AppBundle\Form\Type\UserType;
use Symfony\Component\HttpFoundation\Request;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;

class RegisterController extends Controller
{

    /**
     * @Route("/new", name="user_new")
     */
    public function newAction(Request $request)
    {
        $formType = new UserType(
            $this->get('form.listener.add_state_field'),
            $this->get('form.listener.add_country_field')
        );
    
        $form = $this->createForm($formType, $user = new User());
        $form->handleRequest($request);
        
        if($form->isSubmitted() and $form->isValid()){
            $em = $this->getDoctrine()->getManager();
            
            $em->persist($user);
            $em->flush();
            
            return $this->redirectToAction('alguna acción de la app');
        }
        
        return $this->render('user/new.html.twig', array(
            'form' => $form->createView(),
         ));
    }

}
{% endhighlight %}

### Simplificando el Formulario

Si el formulario va hacer uso de muchos listeners con dependencias, o va a ser utilizado en varias partes de la aplicación, lo mejor es convertir al propio formulario en un servicio:

{% highlight yaml linenos %}
# app/config/services.yml
services:
    form.listener.add_state_field:
        class: AppBundle\Form\Listener\AddStateFieldSubscriber
        arguments: [@security.authorization_checker]
        
    form.listener.add_country_field:
        class: AppBundle\Form\Listener\AddCountryFieldSubscriber
        arguments: [@security.authorization_checker]
        
        
        
    form.type.user:
        class: AppBundle\Form\Type\UserType
        arguments: [@form.listener.add_state_field, @form.listener.add_country_field]
        tags:
            - { name: form.type, alias: user }
{% endhighlight %}

Registramos el `UserType` como servicio y de una vez lo etiquetamos como un `form.type` para facilitar su uso en las distintas partes de la aplicación:

{% highlight php linenos %}
<?php
# src/AppBundle/Controller

namespace AppBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use AppBundle\Entity\User;
use AppBundle\Form\Type\UserType;
use Symfony\Component\HttpFoundation\Request;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;

class RegisterController extends Controller
{
    /**
     * @Route("/new", name="user_new")
     */
    public function newAction(Request $request)
    {
        $form = $this->createForm('user', $user = new User());
        $form->handleRequest($request);
        ...
    }
    
    /**
     * @Route(
     *      "/ajax-form", 
     *      name="user_ajax_form",
     *      conditions="request.isXmlHttpRequest()"
     * )
     */
    public function ajaxFormAction(Request $request)
    {
        $form = $this->createForm('user', $user = new User());
        $form->handleRequest($request);
        ...
    }

}
{% endhighlight %}

Como ven, por medio de los listener se pueden tener formularios extremadamente dinamicos y adaptables a las necesidades de nuestra aplicación.