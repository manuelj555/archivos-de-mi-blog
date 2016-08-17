---

layout: post

title: Pasando Parametros a un Listener de Formulario

---

La idea de este post es demostrar como podemos pasar opciones a un campo de formulario que se crea por medio de un escucha de eventos de formularios, para así dar a conocer el potencial que ofrece el componente de formularios, el despachador de eventos y el OptionsResolver.

Vamos a crear un ejemplo donde tenemos un formulario que muestra un listado de alumnos de una sección seleccionada, donde además solo mostraremos los alumnos con un promedio superior al indicado como parametro al formulario.

Como se puede apreciar, el listado de alumnos depende de la sección que se haya seleccionado, por lo que dicho campo se creará utilizando un listener de formularios. Además, este listener deberá conocer cual es el promedio mínimo que los usuarios deberán poseer para aparecer en el listado.

Este es el código del listener:

```php
<?php

namespace AppBundle\Form\Listener;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;

class AddAlumnosFieldSubscriber implements EventSubscriberInterface
{
	private $promedioMinimo;
    
    public function __construct($promedioMinimo)
    {
        $this->promedioMinimo = $promedioMinimo;
    }
    
    public static function getSubscribedEvents()
    {
        return array(
            FormEvents::PRE_SET_DATA => 'onPreSetData',
            FormEvents::PRE_SUBMIT => 'onPreSubmit',
        );
    }

    public function onPreSetData(FormEvent $event)
    {
		$this->addField($event->getForm(), null);
    }

    public function onPreSubmit(FormEvent $event)
    {
        $data = $event->getData();
        $form = $event->getForm();
        
        $seccion = isset($data['seccion']) ? $data['seccion'] : null;
        
        $this->addField($event->getForm(), $seccion);
    }
    
    protected function addField($form, $seccion)
    {
    	$form->add('alumnos', 'entity', [
        	'class' => 'EntidadAlumnos',
            'property' => 'nombreYPromedio',
            'query_builder' => function($alumnosRepository) use($seccion) {
            	return $alumnosRepository->obtenerQueryPorSeccionYPromedioMinimo(
                	$seccion, $this->promedioMinimo 
                    //el uso de $this en un Closure es a partir de php 5.4
                );
            }
        ]);
    }
}
```

Como se puede ver, nuestro listener recibe el promedio mínimo en su constructor, y con el, añade el campo de alunmos al formulario aplicando los filtros necesarios en la consulta a la base de datos.

Ahora el formulario que agrega el campo de la sección y de alumnos:

```php
<?php

namespace AppBundle\Form\Type;

use AppBundle\Form\Listener\AddAlumnosFieldSubscriber;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolverInterface;


/**
 * @author Manuel Aguirre <programador.manuel@gmail.com>
 */
class PromediosType extends AbstractType
{

    public function getName()
    {
        return 'promedio_alumnos';
    }

    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder->add('seccion', 'entity', [
         	// Configuración del campo...
        ]);

        $builder->addEventSubscriber(new AddAlumnosFieldSubscriber(
        	$options['promedio_minimo'] // Obtenemos este valor como una opción del form.
        ));
    }

    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults([
        	// Opciones por defecto del formulario
        ]);
        
        $resolver->setRequired(['promedio_minimo']);
        
        $resolver->setAllowedTypes([
        	// Solo aceptamos valores flotantes para esta opción.
        	'promedio_minimo' => 'float',
        ]);
    }
}
```

Haciendo uso del componente OptionsResolver disponible en los formularios, creamos una opción llamada `promedio_minimo` y luego hacemos llegar el valor definido en dicha opción al `AddAlumnosFieldSubscriber` en su creación.

Por último le hacemos llegar la opción en el controlador al momento de crear el formulario:

```php
<?php
/... Algún Controlador ...
/**
 * @Route("/una-ruta", name="nombre_ruta")
 */
public function nombreAction(Request $request)
{
    $form = $this->createForm(new AlumnosType(), null, [
    	'promedio_minimo' => 10
    ]);
    $form->handleRequest($request);

    //...
}
```

Como ven el componente de formularios es muy flexible y nos permite crear formularios con alto grado de complejidad gracias a los escuchas de eventos y a el componente OptionsResolver, mediante el cual definimos nuestras propias opciones para los formularios.