---
layout: post
title: Creando un Foro con el MicroKernelTrait de Symfony (Parte 1)
---

En esta ocasión voy a escribir una serie de artículos en donde se describirá paso a paso el desarrollo de una aplicación muy simple utilizando el [MicroKernelTrait de Symfony](http://symfony.com/doc/current/configuration/micro_kernel_trait.html), la cual será un Foro donde los usuarios podrán hacer preguntas y responder las preguntas de otros usuarios.

En esta primera parte sólo vamos a instalar y configurar Symfony con el MicroKernelTrait.

_**Nota**: Los posts se enfocarán a usuarios que tengan conocimientos previos de Symfony, que conozcan sobre el AppKernel, los directorios básicos de un proyecto symfony y de los archivos de configuración y rutas._

- - -

# La aplicación

Para no complicar la aplicación no se tendrán categorías ni se aprobaran preguntas o respuestas, además estás serán las líneas a seguir para el desarrollo de la app:

 * No habrán niveles/perfiles de usuario.
 * Cualquier usuario logueado podrá crear una pregunta.
 * Cualquier usuario logueado podrá responder una pregunta.
 * El usuario que creó la pregunta podrá en cualquier momento dar por resuelta su inquietud.
 * No habrá registro de usuarios (Existirán unos cuantos usuarios de prueba).
 * Habrá un listado de las preguntas del usuario.

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

```json
{
    "require": {
        "symfony/symfony": "^3.1",
        "sensio/framework-extra-bundle": "^3.0",
        "doctrine/dbal": "^2.5"
    },
    "autoload": {
        "psr-4": {
            "": "src/"
        }
    }
}
```

Seguidamente abrimos una consola y nos dirigimos a la carpeta recién creada para proceder a ejecutar:

{% highlight bash %}
php composer install
{% endhighlight %}

Se va a trabajar con tres dependencias para efectos del proyecto:

 * Los componentes de symfony.
 * Las funcionalidades adicionales para los bundles (FrameworkExtraBundle).
 * La librería DBAL de Doctrine (Database Abstraction Layer).

#### Archivos del proyecto

Debemos crear varios ficheros para poder ejecutar la aplicación, la estructura inicial de la que dispondremos es esta:

{% highlight text %}
app/AppKernel.php
app/config/config.yml
app/Resources/views/base.html.twig
app/Resources/views/home/index.html.twig
src/App/Controller/HomeController.php
web/css/foro.css
web/index.php
{% endhighlight %}

A continuación se describe el contenido de cada fichero:

__app/AppKernel.php__

```php
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
         * y Mejoras para controladores y demás.
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
```

__app/config/config.yml__

```yaml
framework:
    secret: StringAleatorio # Cambiar esto
    templating:
        engines: ['twig']
    profiler: { only_exceptions: false }
```

__web/index.php__

```php
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
```

__src/App/Controller/HomeController.php__

```php
<?php

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;

class HomeController extends Controller
{
    /**
     * @Route("/home")
     */
    public function indexAction()
    {
        return $this->render('home/index.html.twig');
    }
}
```

__app/Resources/views/base.html.twig__

{% raw %}
```html
<!doctype html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Micro Foro - Symfony</title>
    {% block stylesheets %}
        <link rel="stylesheet" href="{{ asset('css/foro.css') }}">
    {% endblock stylesheets %}
</head>
<body>
<header>
    <div class="container">
        <div class="header-title">
            Symfony Micro-Foro
        </div>
    </div>
</header>
<article class="container">

    <h1 class="page-header">{% block page_header %}{% endblock %}</h1>

    <section class="content">
        {% block content %}{% endblock %}
    </section>
</article>
</body>
</html>
```
{% endraw %}

__app/Resources/views/home/index.html.twig__

{% raw %}
```html
{% extends 'base.html.twig' %}

{% block content %}
Hola Mundo!
{% endblock %}
```
{% endraw %}

__web/css/foro.css__

{% highlight css %}
/*! normalize.css v4.1.1 | MIT License | github.com/necolas/normalize.css */html{font-family:sans-serif;line-height:1.15;-ms-text-size-adjust:100%;-webkit-text-size-adjust:100%}body{margin:0}article,aside,details,figcaption,figure,footer,header,main,menu,nav,section,summary{display:block}audio,canvas,progress,video{display:inline-block}audio:not([controls]){display:none;height:0}progress{vertical-align:baseline}template,[hidden]{display:none}a{background-color:transparent;-webkit-text-decoration-skip:objects}a:active,a:hover{outline-width:0}abbr[title]{border-bottom:0;text-decoration:underline;text-decoration:underline dotted}b,strong{font-weight:inherit}b,strong{font-weight:bolder}dfn{font-style:italic}h1{font-size:2em;margin:.67em 0}mark{background-color:#ff0;color:#000}small{font-size:80%}sub,sup{font-size:75%;line-height:0;position:relative;vertical-align:baseline}sub{bottom:-0.25em}sup{top:-0.5em}img{border-style:none}svg:not(:root){overflow:hidden}code,kbd,pre,samp{font-family:monospace,monospace;font-size:1em}figure{margin:1em 40px}hr{box-sizing:content-box;height:0;overflow:visible}button,input,optgroup,select,textarea{font:inherit;margin:0}optgroup{font-weight:bold}button,input{overflow:visible}button,select{text-transform:none}button,html [type="button"],[type="reset"],[type="submit"]{-webkit-appearance:button}button::-moz-focus-inner,[type="button"]::-moz-focus-inner,[type="reset"]::-moz-focus-inner,[type="submit"]::-moz-focus-inner{border-style:none;padding:0}button:-moz-focusring,[type="button"]:-moz-focusring,[type="reset"]:-moz-focusring,[type="submit"]:-moz-focusring{outline:1px dotted ButtonText}fieldset{border:1px solid silver;margin:0 2px;padding:.35em .625em .75em}legend{box-sizing:border-box;color:inherit;display:table;max-width:100%;padding:0;white-space:normal}textarea{overflow:auto}[type="checkbox"],[type="radio"]{box-sizing:border-box;padding:0}[type="number"]::-webkit-inner-spin-button,[type="number"]::-webkit-outer-spin-button{height:auto}[type="search"]{-webkit-appearance:textfield;outline-offset:-2px}[type="search"]::-webkit-search-cancel-button,[type="search"]::-webkit-search-decoration{-webkit-appearance:none}::-webkit-input-placeholder{color:inherit;opacity:.54}::-webkit-file-upload-button{-webkit-appearance:button;font:inherit}*{box-sizing:border-box;font-family:sans-serif}body{font-size:14px}header{padding:30px 0;margin:0;background-color:#82b440;box-shadow:5px 10px 10px #555}header .header-title{color:white;font-size:3em}.container{margin:0 auto;padding:0 10px;width:800px}.container .page-header{border-bottom:1px solid #ccc;padding:20px 0;font-size:2.4em;margin-bottom:40px}.container .page-header .button{font-size:.48em;vertical-align:middle}.question{margin-bottom:30px;border-bottom:1px solid #ccc}.question h3{font-size:1.2em;margin:6px 0}.question h3 a{text-decoration:none;color:#4e6e2f}.question .question-content{text-indent:20px;text-align:justify;margin-bottom:10px}.question .question-footer{display:none}.question:last-child{border-bottom:0}.pull-right{float:right}.button,input[type="button"],input[type="submit"],input[type="reset"]{padding:8px;cursor:pointer;text-decoration:none;color:#222;border:1px solid #aaa;border-radius:4px;box-shadow:2px 2px 2px #aaa;display:inline-block;font-size:1.1em}.button.primary,input[type="button"].primary,input[type="submit"].primary,input[type="reset"].primary{color:white;background-color:#82b440;border-color:#5e7f30}.button.primary:hover,input[type="button"].primary:hover,input[type="submit"].primary:hover,input[type="reset"].primary:hover{background:#75a43d}.button:hover,input[type="button"]:hover,input[type="submit"]:hover,input[type="reset"]:hover{background:#f1f1f1}.button:active:hover,input[type="button"]:active:hover,input[type="submit"]:active:hover,input[type="reset"]:active:hover{transform:translateX(1px) translateY(1px)}input[type="submit"]{color:white;background-color:#82b440;border-color:#5e7f30}input[type="submit"]:hover{background:#75a43d}form .form-row{margin-bottom:20px}form .form-row:last-child{margin-bottom:30px}form label{display:block;font-weight:bold;margin-bottom:4px}form .form-widget{width:100%;padding:4px;line-height:25px;border-radius:4px;border:2px solid #5e7f30;box-shadow:2px 2px 4px #aaa}form .form-widget:focus{box-shadow:2px 2px 8px #555}form .description-widget{min-height:200px}form .has-error label{color:#ae0000}form .has-error .form-widget{border-color:#ae0000}form .form-errors{color:#ae0000}
{% endhighlight %}

Listo!, con estos archivos ya podemos correr nuestra aplicación y verificar que todo funcione correctamente.

### Ejecutando el proyecto:

Para ejecutar el proyecto vamos hacer uso del servidor interno de php. Para lo cual nos vamos a colocar en la raiz del proyecto desde la consola y vamos a ejecutar:

	php -S localhost:8000 -t web

Si ahora vamos desde un navegador a `http://localhost:8000/home` deberíamos tener como resultado el mensaje

	Hola Mundo!

Por ahora es todo! ya tenemos el proyecto configurado y funcionando con una página muy simple.

Para la segunda parte vamos a comenzar a crear la base de datos, los modelos y configurar algunos controladores