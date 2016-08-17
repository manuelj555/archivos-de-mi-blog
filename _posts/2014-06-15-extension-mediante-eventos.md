---
layout: post
title: Extendiendo Funcionalidades con el Patron Observador
---

Uno de los principios de SOLID es el de Open-Closed (Abierto-Cerrado abierto a extension, cerrado a modificación).

Muchas veces en nuestras aplicaciones nos encontramos con rutinas que luego de creadas, y de ya estar funcionales, necesitan realizar nuevas tareas como enviar emails, actualizar o crear registros, en fin, tareas que pueden llegar a cambiar o extender cierto proceso.

Estar modificando el código de dichos procesos rompe con un principio básico de [SOLID (object-oriented design)](http://es.wikipedia.org/wiki/SOLID_%28object-oriented_design%29), y es el de "cerrado a modificación", ya que la idea es no tener que tocar el código original de la rutina para agregar funcionalidad.

Esto muchas veces es dificil de lograr, ya que aunque nuestros códigos funcionan muy bien para el objetivo que fueron creados, no son capaces de permitir agregar funcionalidades sin ser modificados.

Veamos un ejemplo de código, tenemos una rutina que aprueba una compra:

{% highlight php linenos linenos %}
<?php

$compra = $model->find(13); //buscamos un registro en la bd por su id

$compra->setStatus(Compra:STATUS_APPROVED); //actualizamos el status a aprovado

$compra->save(); //actualizamos la compra
{% endhighlight %}

Nuestro código se encarga simplemente de actualizar un campo de un registro en la bd, esa es todo su objetivo, y lo hace muy bien.

Pero supongamos que luego nos piden enviar un correo al usuario que hizo la compra, al momento de ser esta aprobada. Generalmente hacemos algo como lo siguente:

{% highlight php linenos linenos %}
<?php

$compra = $model->find(13); //buscamos un registro en la bd por su id

$compra->setStatus(Compra:STATUS_APPROVED); //actualizamos el status a aprovado

$compra->save(); //actualizamos la compra
//codigo añadido
$toUser = $compra->getUser()->getEmail();
$mailer->send("Compra Aprobada", $toUser);
{% endhighlight %}

Ahora nos piden, que solo un usuario administrador pueda aprobar la compra:

{% highlight php linenos linenos linenos %}
<?php

if( $usarioActual->getPerfil() === 'PERFIL_ADMIN' ){

    $compra = $model->find(13); //buscamos un registro en la bd por su id

    $compra->setStatus(Compra:STATUS_APPROVED); //actualizamos el status a aprovado

    $compra->save(); //actualizamos la compra
    //codigo añadido
    $toUser = $compra->getUser()->getEmail();
    $mailer->send("Compra Aprobada", $toUser);
}
{% endhighlight %}

Como se puede ver, cada vez que nos piden agregar cierta funcionalidad ó validaciones, debemos tocar el código original del proceso de aprobación de compras, lo cual lo hace cada vez más complicado e inmantenible. Y estos son agregados faciles de implementar, nos pueden pedir cosas más complejas como verificar que se cumplan ciertas condiciones en la comprar para aprobarla, o que solo se aprueben en ciertos horarios, todo esto produce muchos cambios en nuestro código.

Implementando Observadores
---

Vamos a implementar ahora el patron Observador, haciendo uso del componente [EventDispatcher]({% post_url 2014-03-09-despachando-eventos %}) de symfony. La idea es disparar algunos eventos en el proceso de aprobación de compras, para que luego, si debemos realizar tareas adicionales, las mismas sean ejecutadas en listeners de nuestros eventos.

{% highlight php linenos linenos %}
<?php

$compra = $model->find(13); //buscamos un registro en la bd por su id

$event = PreApproveCompraEvent($compra);

$eventDispatcher->dispatch('compra.pre_aprobacion', $event);

if(!$event->aprobacionCancelada()){
    $compra->setStatus(Compra:STATUS_APPROVED); //actualizamos el status a aprovado    
    $compra->save(); //actualizamos la compra

    $event = ApprovedCompraEvent($compra, $compra->getUser());
    $eventDispatcher->dispatch('compra.aprobacion', $event);
}
{% endhighlight %}

Como ven hemos añadido dos eventos al proceso de aprobación:

*   **compra.pre_aprobacion:** se ejecuta antes de realizar la aprobación, envia a los listeners la compra que se pretende aprobar, y allí podemos entre otras cosas cancelar el proceso de aprobación.
*   **compra.aprobacion:** se ejecuta luego de aprobar, y podemos enviar correos, entre otras cosas.

Validando el Perfil
----

{% highlight php linenos linenos %}
<?php
//esta clase se debe registrar como listener en algún lado, ver la doc de symfony para lograrlo
class ValidarPerfil
{
    protected $sessionUser; //de alguna forma obtenemos el usuario conectado

    public function onPreAprobacion(PreApproveCompraEvent $event)
    {
        if($this->sessionUser->getPerfil() !== 'PERFIL_ADMIN'){
            $event->cancelarAprobacion();
            //como ven, hemos añadido una validacion del perfil sin modificar el código de aprobación
        }
    }
}
{% endhighlight %}

Ahora, gracias al listener, cuando el proceso de aprobación dispare el evento **compra.pre_aprobacion**, se ejecutará el listener, y si el perfil no es **PERFIL_ADMIN**, se llamará a un metodo de la clase event para avisar que hemos cancelado la aprobación.

Enviando Emails
----

{% highlight php linenos linenos %}
<?php
//esta clase se debe registrar como listener en algún lado, ver la doc de symfony para lograrlo
class EnvioEmailAprobacion
{
    public function onPreAprobacion(ApprovedCompraEvent $event)
    {
        $user = $event->getUser();

        $this->mailer->send("Compra Aprobada", $user->getEmail());
    }
}
{% endhighlight %}

Como se puede ver, gracias a un despachador de eventos, y usando el patrón Observador, podemos extender las funcionalidades de nuestros procesos de manera óptima.

En el siguiente post veremos un [Ejemplo en Symfony :)]({% post_url 2014-08-26-extension-mediante-eventos-cambio-status %})