---
layout: post
title: El componente EventDispatcher de Symfony
draft: true
---
Este componente permite implementar el [Patrón Observador](http://es.wikipedia.org/wiki/Observer_%28patr%C3%B3n_de_dise%C3%B1o%29) mediante el uso de una clase (**EventDispatcher**) que contiene a los escuchas de eventos y los llama cuando se dispara algún evento en particular.

En symfony este patrón forma parte importante del [nucleo del framework](http://gitnacho.github.io/symfony-docs-es/book/internals.html#eventos), proveyendo de una serie de [eventos](http://gitnacho.github.io/symfony-docs-es/book/internals.html#eventos) que son disparados durante la ejecución de una petición.

Gracias a ello, mediante el uso de Observadores (Escuchas de Eventos o Listeners), podemos realizar ciertas tareas en cada etapa de la ejecución de un petición en symfony framework, como por ejemplo indentificar la ruta que concuerda con la url actual, verificar si esa ruta está protegida por un firewall y activar el sistema de seguridad en tal caso, entre otros.

El potencial que brinda el uso de este patrón, es muy amplio, ya que dá la posibilidad de extender de manera impresionante una funcionalidad, sin tener que modificar el código del proceso que dispara el evento, solo agregando escuchas como para hacer logs, calculos, enviar correos, he infinidad de cosas para un evento particular.

Como en este post voy es a hacer un ejemplo del uso del componente, hago referencia a sitios donde encontrar documentación al respecto:

*   [Despachador de eventos](http://gitnacho.github.io/symfony-docs-es/components/event_dispatcher/index.html)
*   [The EventDispatcher Component](http://symfony.com/doc/current/components/event_dispatcher/introduction.html)

## Instalando el Componente

Lo primero será descargar el componente EventDispatcher de Symfony, lo recomendable es hacerlo usando composer, y será este el método que usaré para este post.

Aunque este componente se puede usar sin ninguna dependencia, mi consejo es usarlo de la mano del componente [HttpFoundation]({% post_url 2014-02-24-usando-http-foundation %}), y el componente **Debug** que nos brinda mucha información cuando ocurren excepciones.

Acá hay un ejemplo de como debemos tener el composer.json:

{% highlight json linenos %}
{
    "require": {
        "symfony/event-dispatcher": "*"
    }
}
{% endhighlight %}

Ahora solo hace falta ejecutar el comando:

    php composer install

## Como usarlo:

#### La clase EventDispatcher

Esta clase es la encargada de contener y llamar a los escuchas cuando se dispara/ejecuta un evento, es a traves de ella, que registraremos y eliminaremos las funciones o los metodos que estarán escuchando eventos, y es con esta misma clase que invocaremos la ejecución de cada evento a ejecutar.

{% highlight php linenos %}
<?php 

use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();

$dispatcher->addListener('nombre_del_evento', function(){
    //codigo a ejecutar cuando se dispare este evento

}); // Agregamos un escucha para el evento nombre_del_evento

$dispatcher->dispatch('nombre_del_evento'); 
//ejecutamos el evento, por lo que la función registrada previamente, será llamada.
{% endhighlight %}

Para este post, no me extenderé mucho, ya que hay suficiente documentación tanto en ingles como en español, en los links que listé al comienzo de la página.

Lo que he hecho es dar una introducción al uso de este componente para más adelante agregarlo al proyecto que usa [Inyección de dependencias]({% post_url 2014-03-02-inyeccion-de-dependencias-extensiones %})