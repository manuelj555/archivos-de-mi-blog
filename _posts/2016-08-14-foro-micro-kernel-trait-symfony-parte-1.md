---
layout: post
title: Creando un Foro con el MicroKernelTrait de Symfony (Parte 1)
---

En esta ocasión voy a escribir una serie de artículos en donde se describirá paso a paso el desarrollo de una aplicación muy simple utilizando el [MicroKernelTrait de Symfony](http://symfony.com/doc/current/configuration/micro_kernel_trait.html), la cual será un Foro donde los usuarios podrán hacer preguntas y responder las preguntas de otros usuarios.

En esta primera parte solo vamos a instalar y configurar Symfony con el MicroKernelTrait.

_**Nota**: Los posts se enfocarán a usuarios que tengan conocimientos previos de Symfony, que conozcan sobre el AppKernel, los directorios básicos de un proyecto symfony y de los archivos de configuración y rutas._

- - -

# La aplicación

Para no complicar la aplicación no se tendrán categorías ni se aprobaran preguntas o respuestas, además estás serán las lineas a seguir para el desarrollo de la app:

 * No habrán niveles/perfiles de usuario.
 * Cualquier usuario logueado podrá crear una pregunta.
 * Cualquier usuario logueado podrá responder una pregunta.
 * El usuario que creó la pregunta podrá en cualquier momento dar por resuelta su inquietud.
 * No habrá registro de usuarios (Existirán unos cuantos usuarios de prueba).
 * Habrá un listado de las oportunidades del usuario.

Por otro lado, la aplicación constará de los siguientes módulos:

__Inicio__

Muestra un listado de las preguntas realizadas por los usuarios. En esta página aparece el título de la pregunta, la fecha y un resumen de la descripción de la misma. Las preguntas aparecerán desde las más recientes hasta las más antiguas.

__Detalle de Pregunta__

Muestra en detalle la pregunta realizada con su descripción y la información del usuario, además muestra las respuestas que los usuarios han hecho previamente, y contendrá un formulario para añadir una respuesta a la pregunta.

__Crear Pregunta__

Muestra un formulario para crear una pregunta.

__Mis preguntas__

Muestra un listado de las preguntas realizadas por el usuario logueado. En esta página aparece el título de la pregunta y un resumen de la descripción de la misma. Las preguntas aparecerán desde las más recientes hasta las más antiguas.

- - -

# Preguntas

La pregunta será realizada por un usuario previamente logueado y constará de las siguientes propiedades:

 * **id**: Identificador interno de la pregunta.
 * **title**: Título de la pregunta.
 * **description**: Descripción de la pregunta.
 * **author**: Usuario autor de la pregunta.
 * **createdAt**: Fecha de creación de la pregunta.
 * **resolved**: Un atributo para que indica si está resuelta su duda.

# Respuestas

Las respuestas las harán usuarios previamente logueados y constarán de los siguientes atributos:

 * **id**: Identificador interno de la respuesta.
 * **author**: El usuario que responde.
 * **question**: La pregunta que se está respondiendo.
 * **response**: La descripción de la respuesta.
 * **createdAt**: fecha de creación de la respuesta.

- - -

# Requerimientos:

 * Se debe tener al menos **php >= 5.5** ya que se trabajará con Symfony 3 (Aunque se puede trabajar con Symfony 2.8 para php 5.4, que tambien permite el uso de Traits).
 * Se debe tener [composer instalado](https://getcomposer.org/download/) en la máquina.

### Configurando el Proyecto

Vamos a guiarnos de la [documentación oficial](http://symfony.com/doc/current/configuration/micro_kernel_trait.html) para trabajar con el MicroKernelTrait, por lo que lo primero es crear una carpeta para la aplicación, con un nombre como por ejemplo **micro-foro**.

__Instalación__

Crearemos un fichero `composer.json` con el siguiente contenido:

{% highlight json linenos %}
{
    "require": {
        "symfony/symfony": "^3.1",
        "sensio/framework-extra-bundle": "^3.0"
    },
    "autoload": {
        "psr-4": {
            "": "src/"
        }
    }
}
{% endhighlight %}

Seguidamente abrimos una consola y nos dirigimos a la carpeta recien creada para proceder a ejecutar:

{% highlight bash %}
php composer install
{% endhighlight %}

Tal como indica la documentación oficial.

#### Archivos del proyecto

Debemos crear varios ficheros para poder ejecutar la aplicación, la estructura inicial de la que dispondremos es esta:

	|-- app
    |    |-- AppKernel.php
    |    |-- config
    |          |--- config.yml
    |    |-- Resources
    |          |--- views
    |                |--- home
    |                      |--- index.html.twig
    |-- src
    |    |-- App
    |         |--- Controller
    |                   |------ HomeController.php
    |-- web
         |-- index.php

A continuación se describe el contenido de cada fichero:

__app/AppKernel.php__

{% highlight php linenos %}
<?php

use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\Config\Loader\LoaderInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Kernel;
use Symfony\Component\Routing\RouteCollectionBuilder;
use Doctrine\Common\Annotations\AnnotationRegistry;

// require Composer's autoloader
$loader = require __DIR__.'/../vendor/autoload.php';
// auto-load annotations
AnnotationRegistry::registerLoader(array($loader, 'loadClass'));

class AppKernel extends Kernel
{
    use MicroKernelTrait;

    public function registerBundles()
    {
    	/* Inicialmente vamos a trabajar con 3 bundles:
         * El Propio Framework, Twig
         * y Mejoras para controladores y dem�s.
         */
        $bundles = array(
            new Symfony\Bundle\FrameworkBundle\FrameworkBundle(),
            new Symfony\Bundle\TwigBundle\TwigBundle(),
            new Sensio\Bundle\FrameworkExtraBundle\SensioFrameworkExtraBundle(),
        );

        return $bundles;
    }

    protected function configureContainer(ContainerBuilder $c, LoaderInterface $loader)
    {
        $loader->load(__DIR__.'/config/config.yml');
    }

    protected function configureRoutes(RouteCollectionBuilder $routes)
    {
        // Cargamos los controllers usando anotaciones
        $routes->import(__DIR__.'/../src/App/Controller/', '/', 'annotation');
    }
}
{% endhighlight %}

__app/config/config.yml__

{% highlight yaml linenos %}
framework:
    secret: StringAleatorio # Cambiar esto
    templating:
        engines: ['twig']
    profiler: { only_exceptions: false }
{% endhighlight %}

__web/index.php__

{% highlight php linenos %}
<?php

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Debug\Debug;

require __DIR__.'/../app/AppKernel.php';

Debug::enable();

$kernel = new AppKernel('dev', true);
$request = Request::createFromGlobals();
$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response);
{% endhighlight %}

__src/App/Controller/HomeController.php__

{% highlight php linenos %}
<?php

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;

class HomeController extends Controller
{
    /**
     * @Route("/", name="home")
     */
    public function indexAction()
    {
        return $this->render('home/index.html.twig');
    }
}
{% endhighlight %}

__app/Resources/views/home/index.html.twig__

{% highlight jinja linenos %}
{% raw %}
Hola Mundo!
{% endraw %}
{% endhighlight %}

Listo!, con estos archivos ya podemos correr nuestra aplicación y verificar que todo funcione correctamente.

### Ejecutando el proyecto:

Para ejecutar el proyecto vamos hacer uso del servidor interno de php. Para lo cual nos vamos a colocar en la carpeta web del proyecto desde la consola y vamos a ejecutar:

	php -S localhost:8000

Si ahora vamos desde un navegador a `http://localhost:8000/` deberíamos tener como resultado el mensaje

	Hola Mundo!

Por ahora es todo! ya tenemos el proyecto configurado y funcionando con una página muy simple.

Para la segunda parte vamos a comenzar a crear los modelos y configurar algunos controladores