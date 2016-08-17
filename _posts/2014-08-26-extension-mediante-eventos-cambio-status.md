---
layout: post
title: Usando el Event Dispatcher de Symfony con un Ejemplo Práctico
---

Continuando con el post sobre el uso de un [Despachador de Eventos]({% post_url 2014-06-15-extension-mediante-eventos %}), crearemos un ejemplo que permita mostrar como podemos usarlo cuando desarrollamos con Symfony.

La idea es aprobar a ciertos usuarios en una aplicación luego de que estos se han registrado, además, al realizar la aprobación se quiere que le llegue un correo a dicho usuario informandole que su cuenta ha sido habilitada.

La clase User
----

Comenzaremos con una clase que será nuestro modelo o entidad, el código de la misma es el siguiente:

{% highlight php linenos %}
<?php

namespace MyBundle\Entity;

class User
{
    protected $name;
    protected $email;
    protected $status;

    // ...getters y setters
}

{% endhighlight %}

La clase UserEvents
----
Esta clase simplemente contendrá constantes que nos ayudarán a documentar los eventos y nos permitirán usar dichas constantes en vez de strings al despachar eventos, lo que ayuda a minimizar los errores al tipear.

{% highlight php linenos %}
<?php

namespace MyBundle;

final class UserEvents
{
    /**
     * Este evento se ejecuta antes de cambiar el estatus del usuario a aprobado
     * Los listener de este evento deben esperar una instancia de:
     * 
     * Symfony\Component\EventDispatcher\GenericEvent
     * 
     * Si alguno de los listener cancela la propagación del 
     * evento ($event->stopPropagation()), la aprobación
     * no se realiza, ni se llama al evento post_approve.
     */
    const PRE_APPROVE = 'my_bundle.user.pre_approve';

    /**
     * Este evento se ejecuta despues de cambiar el estatus del usuario a aprobado
     * Los listener de este evento deben esperar una instancia de:
     * 
     * Symfony\Component\EventDispatcher\GenericEvent
     * 
     * Si en el evento pre_aprove, se cancela la propagación de 
     * dicho evento ($event->stopPropagation()),
     * el evento post_approve no es disparado.
     */
    const POST_APPROVE = 'my_bundle.user.post_approve';
}

{% endhighlight %}

La clase UserManager
----
Es una buena práctica crear un manager para nuestros modelos, y así no tener la lógica de los mismos directo en los controladores (recordemos: controladores flacos, modelos gordos).

Para efectos de este ejemplo, nuestro manager solo tendrá un método relevante para el manejo de los usuarios, el mismo tendrá por nombre **approve** y esperará una instancia de **MyBundle\Entity\User** que será el usuario que aprobaremos:

{% highlight php linenos %}
<?php

namespace MyBundle\Model;

use MyBundle\UserEvents;
use MyBundle\Entity\User;
use Doctrine\ORM\EntityManager;
use Symfony\Component\EventDispatcher\GenericEvent;
use Symfony\Component\EventDispatcher\EventDispatcherInterface;

class UserManager
{
    protected $em;

    protected $dispatcher;

    public function __construct(EntityManager $em, EventDispatcherInterface $dispatcher)
    {
        $this->em = $em;
        $this->dispatcher = $dispatcher;
    }

    public function approve(User $user)
    {
        $event = new GenericEvent($user);
        $this->dispatcher->dispatch(UserEvents::PRE_APPROVE, $event);

        if ($event->isPropagationStopped()) {
            return false; //cancelamos la aprobación
        }

        $user->setStatus(User::STATUS_APPROVED);
        $this->em->persist($user);
        $this->em->flush();

        $event = new GenericEvent($user);
        $this->dispatcher->dispatch(UserEvents::POST_APPROVE, $event);

        return true;
    }
}

{% endhighlight %}

Como se puede ver, el código del método **approve** es bastante simple, dispara dos eventos y en medio de los mismos ejecuta el cambio de estatus y lo persiste en la BD. 

Los eventos que ejecutamos son **my\_bundle.user.pre\_approve** y **my\_bundle.user.post\_approve**, y le pasamos una instancia de **[Symfony\Component\EventDispatcher\GenericEvent](http://symfony.com/doc/current/components/event_dispatcher/generic_event.html)**. Pudimos haber creado una clase Event propia, pero como solo pasaremos el objeto $user, no hace falta, para casos donde queramos pasar más objetos, o tener mejor control de los eventos, podemos crearnos nuestras clases Event personalizadas.

Registrando el UserManager en el Container
----

Nuestro UserManager necesita que se le pasen dos objetos para realizar sus tareas de aprobación, estos son el entity manager de doctrine y el event dispatcher de symfony, para hacer esto, registraremos nuestra clase como un servicio en el contenedor y le inyectamos los servicios/objetos que necesita:

{% highlight yaml linenos %}
#  MyBundle/Resources/config/services.yml
services:
    my_bundle.user_manager:
        class: MyBundle\Model\UserManager
        arguments:
            - @doctrine.orm.default_entity_manager
            - @event_dispatcher
{% endhighlight %}

Con esto ya tenemos registrado nuestro manager como servicio, y podemos acceder a el por medio del id **my\_bundle.user\_manager**.

El controlador
----
Ahora creamos nuestra acción en algún controlador, para que se realize el proceso de aprobación:

{% highlight php linenos %}
<?php

namespace MyBundle\Controller;

use MyBundle\Entity\User;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\ParamConverter;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class UserController extends Controller
{
    /**
     * @ParamConverter("user", class="MyBundle:user") usamos anotaciones :)
     *
     * @link http://symfony.com/doc/master/bundles/SensioFrameworkExtraBundle/annotations/converters.html
     */
    public function approveAction(User $user)
    {
        if($this->get('my_bundle.user_manager')->approve($user)){
            // enviamos un flash por ejemplo
        }

        return $this->redirect(....);
    }
}
{% endhighlight %}

Nuestro controlador ha quedado muy simple, ya que la mayor parté del código (lógica de negocio) se encuentra en el user_manager.

Y el correo?
-----
Notarán que el método **approve** de la clase **UserManager** no realiza el envío de correo al aprobar al usuario, esto es porque esta tarea se la dejaremos a un listener que crearemos a continuación.

El Listener para el Correo
----

{% highlight php linenos %}
<?php

namespace MyBundle\Listener;

use Symfony\Component\EventDispatcher\GenericEvent;

class SendApprovedEmailListener
{
    protected $mailer;

    public function __construct($mailer)
    {
        $this->mailer = $mailer;
    }

    /**
     * Este método será el encargado de enviar el correo electrónico luego de
     * que el usuario haya sido aprobado.
     *
     * Para más info sobre GenericEvent ver:
     * @link http://symfony.com/doc/current/components/event_dispatcher/generic_event.html
     */
    public function onPostApprove(GenericEvent $event)
    {
        $user = $event->getSubject(); //nos devuelve el objeto User

        $message = \Swift_Message::newInstance()
            ->setSubject('Cuenta Aprobada!')
            ->setFrom($from) //lo sacamos de algún lado (container, bd, ...)
            ->setTo($user->getEmail())
            ->setBody($body); //lo sacamos de algún lado (bd, twig, ...)

        $this->mailer->send($message);
    }
}
{% endhighlight %}

Ya tenemos nuestro listener creado, ahora debemos registrarlo en el contenedor y agregarle las etiquetas que lo identifiquen como un escucha de eventos:

{% highlight yaml linenos %}
#  MyBundle/Resources/config/services.yml
services:
  my_bundle.user_manager:
   ....

  my_bundle.listener.user.send_approved_email:
    class: MyBundle\Listener\SendApprovedEmailListener
    arguments:
        - @mailer
    tags:
      - {name: kernel.event_listener, event: my_bundle.user.post_approve, method: onPostApprove}
{% endhighlight %}

Listo!!!. Gracias a la etiqueta **kernel.event\_listener** de symfony, nuestra clase está escuchando el evento **my\_bundle.user.post\_approve**, y cuando el mismo sea disparado en el UserManager al aprobar, el método **onPostApprove** del listener será invocado y se enviará el correo. Todo esto sin haber tenido que modificar el código de aprobación de usuarios.

Espero que este ejemplo sirva para que de ahora en adelante aprovechemos mejor las ventajas que brinda usar el despachador de eventos de symfony, ya que así tendremos la posibilidad de crear códigos muy simples y extensibles de manera elegante y sencilla.

Más adelante crearemos un listener que verifique el estatus del usuario antes de aprobarlo, ya que por ejemplo, si un usuario fué previamente aprobado, o rechazado, no debería poderse aprobar.

Parte 2	[Despachador de Eventos (Ejemplo Practico 2)]({% post_url 2014-09-15-extension-mediante-eventos-cambio-status-2 %})