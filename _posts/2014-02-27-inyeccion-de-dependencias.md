---
layout: post
title: El componente DependencyInjection
---
Creo que ya existe en la web bastante información de como el framework symfony implementa la inyección de dependencias, definiendo clases llamadas **servicios** que se encuentran disponibles en un contenedor.

*   [Inyección de dependencias](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/)
*   [El contenedor de servicios](http://librosweb.es/symfony_2_3/capitulo_16.html)

Pero acá vamos a exponer como implementar el componente en un proyecto PHP cualquiera.

## Instalando el Componente

Lo primero será descargar el componente DependencyInjection de Symfony, lo recomendable es hacerlo usando composer, y será este el método que usaré para este post.

Aunque este componente se puede usar sin ninguna dependencia, mi consejo es usarlo de la mano del componente [HttpFoundation]({% post_url 2014-02-24-usando-http-foundation %}), el componente **Config** que nos brindará la posibilidad de cachear las definiciones de servicios y leerlas a partir de archivos yaml y el componente **Yaml**, que será quien parsee los datos de yml a php.

Acá hay un ejemplo de como debemos tener el composer.json:

{% highlight json linenos %}
{
    "require": {
        "symfony/http-foundation": "*",
        "symfony/dependency-injection": "*",
        "symfony/yaml": "*",
        "symfony/config": "*",
        "symfony/routing": "*"
    }
}
{% endhighlight %}

El routing lo usaremos para probar la definción de servicios.

Ahora solo hace falta ejecutar el comando:

    php composer install

#### Organizando el código

Vamos ahora a crear una serie de carpetas para organizar los archivos, se me ocurrió algo como:

    proyecto
        |---app
        |    |---cache
        |    |---config
        |    |     |---services.yml
        |    |---bootstrap.php
        |
        |---vendor

Esta estructura de archivos es igual a la del ejemplo del uso de **[Componente Routing]({% post_url 2014-02-26-symfony-routing %})**

La función de cada carpeta es la siguiente:

Carpetas:

*   **cache** acá irán los archivos cacheados
*   **config** acá irán los yml con las definiciones de los servicios
*   **vendors** las libs instaladas mediante composer

Archivos:

*   **app/config/services.yml** la definción de los servicios
*   **public/bootstrap.php** configuracion del componente

## Como funciona el Componente


Este componente contiene varias clases principales que realizan los procesos de creación y obtención de los servicios, algunas de estas clases son:

#### ContainerBuilder

Esta clase es la que contiene las definiciones, y los parametros, permite modificarlos y crear las instancias para ser usadas posteriormente en la aplicación:

{% highlight php linenos %}
<?php

// Recordemos haber cargado el autoload de composer antes de usar los componentes.

use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container->register('request_context', 'Symfony\\Component\\Routing\\RequestContext');
// Registramos la clase Symfony\Component\Routing\RequestContext 
// y le asignamos el id request_context

$container->compile(); //prepara todas las definiciones para su uso

$requestContext = $container->get('request_context'); //devolvemos/creamos la instancia
{% endhighlight %}

Más info por acá:

*   [Uso Básico del Contenedor](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/introduction.html#uso-basico)
*   [Tipos de inyección](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/types.html)

###### Cargando los servicios desde YAML

Para hacer más simple el uso del componente vamos a crear y configurar los servicios en archivos **yml**, ya que desde mi punto de vista es más simple que con php, para ello haremos uso del componente **Config** y **Yaml** que instalamos previamente con composer:

{% highlight php linenos %}
<?php //  proyecto/app/bootstrap.php

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

define('APP_PATH', __DIR__ . '/'); //contiene la ruta hasta app
define('DEBUG', true); // true en desarollo y false en producción

$container = new ContainerBuilder();
$loader = new YamlFileLoader($container, new FileLocator(APP_PATH . 'config/'));
$loader->load('services.yml');

$container->compile();

$requestContext = $container->get('request_context');

{% endhighlight %}

Nuestro archivo services.yml

{% highlight yaml linenos %}
#    proyecto/app/config/services.yml
parameters:
    app_name: Prueba del Inyector de Dependencias

services:
    request_context:
        class: Symfony\Component\Routing\RequestContext
{% endhighlight %}

**Es importante crear al menos un parametro en el archivo,o en el contenedor directamente antes de guardar en caché, ya que de lo contrario al cachear, el contenedor da un error del tipo**:

    FatalErrorException: Error: Call to a member function get() on a non-object in D:\wamp\www\pruebas\di\vendor\symfony\dependency-injection\Symfony\Component\DependencyInjection\Container.php line 147

Con esto ya tenemos definido nuestro servicio en yaml.

Pero estár leyendo el archivo yml en cada petición (cuando cresca pueden ser muchos servicios y parametros) y compilar esas definciones, consume mucho rendimiento, por lo que lo mejor es cachear los servicios para solventar esto.

## Cacheando las Definiciones

{% highlight php linenos %}
<?php //  proyecto/app/bootstrap.php


use Symfony\Component\Config\ConfigCache;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Dumper\PhpDumper;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

define('APP_PATH', __DIR__ . '/'); //contiene la ruta hasta app
define('DEBUG', true); // true en desarollo y false en producción

$file = APP_PATH . 'cache/container.php';
$containerConfigCache = new ConfigCache($file, DEBUG);

if (!$containerConfigCache->isFresh()) { //si no está actualizado
    $containerBuilder = new ContainerBuilder();
    $containerBuilder->setParameter('app_path', APP_PATH);
    $containerBuilder->setParameter('debug', DEBUG);

    $loader = new YamlFileLoader($containerBuilder, new FileLocator(APP_PATH . 'config/'));
    $loader->load('services.yml');

    $containerBuilder->compile();

    $dumper = new PhpDumper($containerBuilder);
    $containerConfigCache->write(
            $dumper->dump(array('class' => 'MyCachedContainer')), $containerBuilder->getResources()
    );
}

require_once $file;
$container = new MyCachedContainer();
// ...
//ya podemos usar los servicios :-)
$requestContext = $container->get('request_context');
{% endhighlight %}

Con esto logramos que todos los servicios y parametros se cacheen en una clase contenedora, la gran ventaja de esta clase cacheada es que crea código php equivalente a lo que definimos en el yml, veamos un ejemplo de lo que expongo:

### Viendo el código de la Cache

Creamos la siguiente definición para el servicio router:

{% highlight yaml linenos %}
#    proyecto/app/config/services.yml
parameters:
    router.file: routing.yml
    router.config:
        cache_dir: %app_path%config/
        debug: %debug%
        
services:
    request_context:
        class: Symfony\Component\Routing\RequestContext
        calls:
            - [setBaseUrl, ['ejemplo_de_base_url']]
            - [setHost, ['localhost']]
        
    route.locator:
        class: Symfony\Component\Config\FileLocator
        arguments:
            - ["%app_path%config/"]
        
    route.loader.yml:
        class: Symfony\Component\Routing\Loader\YamlFileLoader
        arguments:
            - @route.locator
        
    router:
        class: Symfony\Component\Routing\Router
        arguments:
            - @route.loader.yml
            - %router.file%
            - %router.config%
            - @request_context
{% endhighlight %}

Al ejecutar la app, se crea el archivo **app/cache/container.php**, y si miramos algunos de sus métodos, encontraremos lo siguiente:

{% highlight php linenos %}
<?php //  proyecto/app/cache/container.php
//...
$this->methodMap = array(
    'request_context' => 'getRequestContextService',
    'route.loader.yml' => 'getRoute_Loader_YmlService',
    'route.locator' => 'getRoute_LocatorService',
    'router' => 'getRouterService',
);
//... Contiene los métodos de la clase a llamar por cada servicio

// algunos métodos:

protected function getRequestContextService()
{
    $this->services['request_context'] = $instance = new \Symfony\Component\Routing\RequestContext();

    $instance->setBaseUrl('ejemplo_de_base_url');
    $instance->setHost('localhost');

    return $instance;
}

protected function getRoute_Loader_YmlService()
{
    return $this->services['route.loader.yml'] = new \Symfony\Component\Routing\Loader\YamlFileLoader($this->get('route.locator'));
}

protected function getRoute_LocatorService()
{
    return $this->services['route.locator'] = new \Symfony\Component\Config\FileLocator(array(0 => 'D:\\wamp\\www\\pruebas\\di\\app/config/'));
}

protected function getRouterService()
{
    return $this->services['router'] = new \Symfony\Component\Routing\Router($this->get('route.loader.yml'), 'routing.yml', array('cache_dir' => 'D:\\wamp\\www\\pruebas\\di\\app/config/', 'debug' => true), $this->get('request_context'));
}

{% endhighlight %}

Como podemos ver es impresionante como el componente cachea las definiciones, por ejemplo el método **getRequestContextService** que aparte de crear la instancia de la clase crea en código php los llamados a los métodos **setBaseUrl** y **setHost** definidos en el yml para que se vea como funciona la cache.

Tambien podemos apreciar en el método **getRouterService** que el arreglo del tercer argumento tiene data estática, y no se buscará ningun parametro externo en su creación, esto da un gran rendimiento.

## Inyectando en contenedor en un servicio

Tambien es posible pasarle a un servicio el contenedor, de la siguiente manera:

{% highlight yaml linenos %}
services:
    mi_servicio:
        class: Mi_Clase_Cualquiera
        arguments:
            - @service_container
          # - otros argumentos

{% endhighlight %}

El container por defecto se agrega como servicio con el id **service_container**, por lo que podemos inyectarlo y usarlo como cualquier otro servicio.

Bueno espero que este post haya sido de ayuda para implementar el componente en aplicaciones php, o al menos ayude a comprender un poco el funcionamiento interno del mismo.