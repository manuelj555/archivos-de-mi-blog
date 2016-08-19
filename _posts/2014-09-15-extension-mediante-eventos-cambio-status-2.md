---
layout: post
title: Usando el Event Dispatcher de Symfony con un Ejemplo Práctico (Parte 2)
---

Continuando con el post sobre el uso de un [Despachador de Eventos (Ejemplo Practico)]({% post_url 2014-08-26-extension-mediante-eventos-cambio-status %}), crearemos ahora un listener que verifique el estatus del usuario antes de aprobarlo, para así evitar que se le reenvie un correo a un usuario previamente aprobado.

Primeramente crearemos el listener para luego registrarlo como un servicio que escucha el evento **my\_bundle.user.pre\_approve**.

El Listener
----

{% highlight php linenos %}
<?php

namespace MyBundle\Listener;

use Symfony\Component\EventDispatcher\GenericEvent;

class VerifyStatusListener
{
    protected $logger;

    public function __construct($logger = null)
    {
        $this->logger = $logger;
    }

    /**
     * Este método será el encargado de verificar el status del usuario para ver
     * si este es aprobable o no.
     *
     * Para más info sobre GenericEvent ver:
     * @link http://symfony.com/doc/current/components/event_dispatcher/generic_event.html
     */
    public function onPreApprove(GenericEvent $event)
    {
        $user = $event->getSubject(); //nos devuelve el objeto User

        if($user->getStatus() == User::STATUS_APPROVED){
            $event->stopPropagation(); //detenemos la ejecucion del evento, y evitamos la aprobacion del usuario
            if($this->logger){
              $this->logger->debug("Se detuvo la aprobación del usuario, ya que este se encuentra aprobado");
            }
        }
    }
}
{% endhighlight %}

Nuestro listener tiene un método **onPreAprove** que lo que hace es verificar que el usuario no tenga el estatus de aprobado, para permitir o evitar su aprobacion.

Además si se detiene la aprobación, enviamos un log, para saber (al menos en desarrollo) porque no se realizó la aprobación.

### StopPropagation

Como se puede observar en el Listener, hemos cancelado la ejecución de la aprobación (y de los demás listeners del evento my\_bundle.user.pre\_approve) llamando al método **$event->stopPropagation()**, el cual está disponible en cualquier objeto de tipo Event cuando usamos el desapachador de eventos de symfony.

## Registrando el Listener

Ya tenemos nuestro listener creado, ahora debemos registrarlo en el contenedor y agregarle las etiquetas que lo identifiquen como un escucha de eventos:

{% highlight yaml linenos %}
#  MyBundle/Resources/config/services.yml
services:
  ...
  my_bundle.listener.user.verify_status:
    class: MyBundle\Listener\VerifyStatusListener
    arguments:
        - @?logger # Optional
    tags:
      - {name: kernel.event_listener, event: my_bundle.user.pre_approve, method: onPreApprove}
{% endhighlight %}

Con esto ya tenemos nuestro listener funcionando, y al ejecutarse el evento, si el usuario se encuentra previamente aprobado, el proceso será cancelado.

Para lograr esto debemos recordar que en la clase UserManagar, en su evento approve, tenemos una condicion:

{% highlight php linenos %}
<?php

  ...

  public function approve(User $user)
  {
      $event = new GenericEvent($user);
      $this->dispatcher->dispatch(UserEvents::PRE_APPROVE, $event);

      // esta condición verifica si algún listener ha detenido la ejecución de la
      // aprobación llamando a $event->stopPropagation()
      if ($event->isPropagationStopped()) {
          return false; //cancelamos la aprobación
      }

      ...
  }
{% endhighlight %}

Bueno, así podemos crear otros listener para verificaciónes y ejecuciones adicionales de procesos.
