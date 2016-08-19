---
layout: post
title: Usando el Routing de Symfony
---
Este componente me ha parecido muy interesante y potente para usar en proyectos PHP, ya que con su implementación se hace muy facil tener un controlador frontal en nuestras aplicaciones. Mediante el manejo de rutas agradables y a la vez potentes, y la asociación de las mismas con el "controlador" a ejecutar, pudiendo ser este último un archivo php, una función, un método de una clase, etc. eso ya queda a nuestra decisión.

En este post mostraré como se puede implementar en una aplicación con controlador frontal y sistema de rutas en php.

## Instalando el Componente

Lo primero será descargar el componente Rounting de Symfony, lo recomendable es hacerlo usando composer, y será este el método que usaré para este post.

Aunque este componente se puede usar sin ninguna dependencia, mi consejo es usarlo de la mano del componente [HttpFoundation]({% post_url 2014-02-24-usando-http-foundation %}), el componente **Config** que nos brindará la posibilidad de cachear las rutas y leerlas a partir de archivos yaml, el componente **Yaml**, que será quien parsee los datos de yml a php y por último y muy importante el componente **Debug** que nos brinda mucha información cuando ocurren excepciones.

Acá hay un ejemplo de como debemos tener el composer.json:

{% highlight json linenos %}
{
    "require": {
        "symfony/http-foundation": "*",
        "symfony/routing": "*",
        "symfony/yaml": "*",
        "symfony/config": "*",
        "symfony/debug": "*"
    }
}
{% endhighlight %}

Ahora solo hace falta ejecutar el comando:

    php composer install 

#### Organizando el código

Vamos ahora a crear una serie de carpetas para organizar los archivos, se me ocurrió algo como:

    proyecto
        |---app
        |    |---cache
        |    |---config
        |    |     |---config.yml
        |    |     |---routing.yml
        |    |---controller
        |    |---views
        |    |---bootstrap.php
        |   
        |---public
        |     |----web.php
        |---vendor

La función de cada carpeta es la siguiente:

Carpetas:

*   **cache** acá irán los archivos cacheados (Las rutas y demás)
*   **config** acá irán los yml con la config y rutas de la app
*   **controller** nuestras clases o archivos controladores
*   **views** nuestros archivos con la presentación visual
*   **public** los assets y el archivo web.php que hará de controlador frontal
*   **vendors** las libs instaladas mediante composer

Archivos:

*   **app/config/config.yml** cualquier configuración que se nos ocurra :-)
*   **app/config/routing.yml** las rutas de la app
*   **public/web.php** controlador frontal de la aplicación.

Esa fue la estructura de archivos que se me ocurrió a mi, tu puedes usar la de tu preferencia.

### Creando el Controlador Frontal

Nuestro controlador frontal será un archivo php (**public/web.php**) que ejecutará todas las peticiones que se hagan a la aplicación, el será el encargado de iniciar las configuraciones necesarias para que la app se ejecute correctamente.

Creamos nuestro archivo con el siguiente código:
{% highlight php linenos %}
<?php // proyecto/public/web.php

require_once __DIR__ . '/../vendor/autoload.php';

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Debug\Debug;

Debug::enable();
//Debug::enable(null, false); //descomentar en producción y comentar el anterior

define('APP_PATH', dirname(__DIR__) . '/app/'); //contiene la ruta hasta app
define('DEBUG', true); // true en desarollo y false en producción

$request = Request::createFromGlobals(); //creamos el objeto Request

{% endhighlight %}

Esto es lo primero que agregamos al archivo, incluimos el autoload de composer y creamos la instancia del Request de la lib HttpFoundation.

## Como funciona el Routing

Este componente contiene varias clases principales que realizan los procesos de creación y obtención de las rutas, aparate de obtener la ruta que coincide con la petición que se está ejecutando, estas clases son:

##### Route

Esta clase contiene la definción de una ruta, la ruta puede ser cualquier cadena de url válida y puede contener parametros variables:
{% highlight php linenos %}
<?php

use Symfony\Component\Routing\Route;

$route = new Route('/paises', array(
    '_controller' => 'MyController::metodo' //este indice se me ocurrió, no es obligatorio
));
//el primer argumento es el path de la ruta
//el segundo es un arreglo con los valores por defecto que formarán parte de la definición de la ruta
//tiene otro argumentos que no voy a exponer, ya que los manejaremos por yml :-)

$route = new Route('/paises/crear', array(
    '_file' => 'paises.php' //otro indice inventado por mi
));

$route = new Route('/paises/editar/{id}', array(
    '_file' => 'paises/editar.php',
));

$route = new Route('/paises/{pagina}', array(
    '_file' => 'paises/listado.php',
    'pagina' => 1, //valor por defecto para el parametro pagina
));


{% endhighlight %}

Lo **recomendable** es que el path siempre comience con un slash **/** aunque no es obligatorio.
El segundo parametro es interesante ya que allí podemos poner los indices que queramos, es decir, que esos valores quedan abiertos para ser usados con la convención que deseemos aplicar en las aplicaciones. Además aqui podemos especificar valores por defecto para las rutas.

Es en ese array donde se especifica la función, archivo y/o clase::metodo a ejecutar para esa ruta en particular, y el indice para especificar dicho valor puede ser el que queramos, aunque la convención que al menos yo he seguido es la de que ese indice comienze con un **_**.

##### RouteCollection

En esta clase se agregan cada una de las rutas que tendrá la aplicación:
{% highlight php linenos %}
<?php

use Symfony\Component\Routing\RouteCollection;
use Symfony\Component\Routing\Route;

$route = new Route('/paises', array('controller' => 'MyController'));
//$route es la clase que contiene la definción de la ruta
$routes = new RouteCollection();
$routes->add('paises_listado', $route); //con add agregamos una ruta y le damos un nombre que debe ser unico

$routes->add('paises_add', new Route('/paises/crear', array('controller' => 'Otro')));
$routes->add('paises_editar', new Route('/paises/editar/{id}', array('controller' => 'Otro')));

{% endhighlight %}

Lo más importante de esta clase es su método **add**, mediante el cual establecemos primero el nombre que identificará a la ruta en la aplicación, y segundo, el objeto con la definicion de la ruta.

##### RequestContext

Esta clase contiene información de la petición, realmente nosotros no la usaremos directamente, sino más bien será el router quien la necesitará (para mi es necesaria para poder usar el componente independientemente).

Se verá más adelante un ejemplo de su uso.

##### UrlMatcher

Esta clase se encarga de obtener la ruta que coincide con el patro de la url de la petición:
{% highlight php linenos %}
<?php

use Symfony\Component\Routing\Exception\ResourceNotFoundException;
use Symfony\Component\Routing\Matcher\UrlMatcher;
use Symfony\Component\Routing\RequestContext;

$routes = ... //instancia de RouteCollection con las rutas del ejemplo anterior

$context = new RequestContext($_SERVER['REQUEST_URI']); //el REQUEST_URI es el baseUrl que usa, es opcional

$matcher = new UrlMatcher($routes, $context);//nuestro matcher usa las rutas y el contexto de la petición

try{
    $parameters = $matcher->match('/paises'); //acá realizamos la busqueda de una ruta que concuerde
    //si se encuentra una ruta, se devuelve el arreglo que pasamos en la defincion de la ruta:
    // array('_file' => 'paises/lista.php')

    $parameters = $matcher->match('/paises/editar/2');
    // array('_file' => 'paises/editar.php', 'id' => 2)
}catch(ResourceNotFoundException $e){
    //cuando no se encuentra ninguna concordancia, se lanza esta excepcion
}

{% endhighlight %}

##### UrlGenerator

Esta clase se encarga de crear una url a partir del nombre de una ruta y los parametros que se pasen, es la clase a usar cuando necesitamos crear links en nuestras páginás o queremos redirigir una petición, etc. Ejemplo de uso:
{% highlight php linenos %}
<?php

use Symfony\Component\Routing\Generator\UrlGenerator;

$routes = ... //instancia de RouteCollection con las rutas del ejemplo anterior
$context = ... //instancia de RequestContext

$generator = new UrlGenerator($routes, $context);

$url = $generator->generate('paises_listado'); //le pasamos el nombre de la ruta
//devuelve: /paises

$url = $generator->generate('paises_editar', array('id' => 3)); //le pasamos el nombre de la ruta
//devuelve: /paises/editar/3
{% endhighlight %}

### Rutas en Yml

Ahora bien al menos para mi es muy engorroso estar creando las rutas con:

    $routes->add('paises_add', new Route('/paises/crear', array('controller' => 'Otro')));

Por lo que me parece que es mejor usar una de las alternativas que ofrece el componente para cargar rutas, estas alternativas son: yaml, php, xml y anotaciones. En mi caso usaré yaml por ser el más simple desde mi punto de vista:
{% highlight php linenos %}
<?php

use Symfony\Component\Config\FileLocator;
use Symfony\Component\Routing\Loader\YamlFileLoader;

$locator = new FileLocator(array(APP_PATH . 'config/')); //encuentra archivos en el dir especificado.
$loader = new YamlFileLoader($locator); //cargador de rutas mediante yml
$collection = $loader->load('routing.yml'); //lee las rutas del yml y devuelve una instancia de RouteCollection con las rutas.
{% endhighlight %}

#### El routing.yml

Ahora nuestras rutas las definimos de la siguiente manera:
{% highlight yaml linenos %}
# proyecto/app/config/routing.yml

paises_listado:
    path: /paises
    defaults:       # los valores de la definición de la ruta
        _file: paises/lista.php

paises_crear:
    path: /paises/crear
    defaults:
        _file: paises/crear.php

paises_editar:
    path: /paises/editar/{id}
    defaults:
        _file: paises/editar.php
    requirements:    # Permite definir restricciones para los parametros
        id: +d       # Solo números :-)
{% endhighlight %}

Ahora vemos lo sencillo que se ha hecho crear rutas gracias al uso de Yml.


#### La clase Router

Como vimos con enterioridad, el componente hace uso de unas cuantas clases para el manejo de rutas, pero además ofrece una clase que nos permite manejar de una forma más sencilla todo esto, ya que ofrece métodos para encontrar rutas y crear urls a partir de esas rutas, tambien se encarga cachear las mismas para mejorar el rendimiento de la aplicación:

```php
<?php

use Symfony\Component\Routing\Router;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\Routing\Loader\YamlFileLoader;

$requestContext = new RequestContext($_SERVER['REQUEST_URI']);

$locator = new FileLocator(array(APP_PATH . 'config/'));
$loader = new YamlFileLoader($locator);

$options = array(
    'cache_dir' => __DIR__.'/cache', //directorio donde serán cacheadas las rutas
    'debug' => true, //solo regenera la caché en debug, en producción debe ser false para mejor rendimiento
);

$router = new Router($loader, 'routes.yml', $options, $requestContext);
//esta clase espera el loader, que puede ser para xml, yml, php ó anotaciones
// espera el archivo que contiene las rutas en un formato acorde al loader usado
// espera un arreglo de opciones
// y espera el requestContext

//el router encuentra la ruta correspondientes a cada request
try{
    $parameters = $router->match('/paises');
    // array('_file' => 'paises/lista.php')

    $parameters = $router->match('/paises/editar/2');
    // array('_file' => 'paises/editar.php', 'id' => 2)
}catch(ResourceNotFoundException $e){
    //cuando no se encuentra ninguna concordancia, se lanza esta excepcion
}

// tambien permite generar url para las rutas

$url = $router->generate('paises_listado');
//              /paises

$url = $router->generate('paises_editar', array('id' => 3));
//              /paises/editar/3
```

Con eso ya hemos visto como funciona el componente.

## Ajá Bueno y Ahora?

Ya vimos muy por encima lo que puede ofrecer el componente routing de symfony, ahora vamos a ver como usarlos en nuestra aplicación.

### Creando el Controlador Frontal (Continuación)

Ya tenemos nuestro archivo **public/web.php** que pronto ampliaremos para agregar el código necesario para el front controller, ahora vamos a crear un archivo en **app/bootstrap.php** para configurar allí todo el componente de las rutas.

Para efectos de este ejemplo los controladores en una primera etapa de la documentación serán archivos php, más adelante veremos como manejarlos mediante clases controladoras.

Las rutas que trabajará el proyecto serán de la forma:

    proyecto/public/web.php
    proyecto/public/web.php/paises
    proyecto/public/web.php/paises/crear
    proyecto/public/web.php/paises/editar/3
    ...

Así todas esas url ejecutarán siempre el archivo public/web.php y la ruta la determinaremos mediante el pathinfo de la petición.

El código del archivo **app/bootstrap.php** es el siguiente:

{% highlight php linenos %}
<?php //proyecto/app/bootstrap.php

use Symfony\Component\Config\FileLocator;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Exception\ResourceNotFoundException;
use Symfony\Component\Routing\Loader\YamlFileLoader;
use Symfony\Component\Routing\RequestContext;
use Symfony\Component\Routing\Router;

//configuración para cargar las vistas
define('VIEW_PATH', APP_PATH . 'views/');
//configuración del router

$requestContext = new RequestContext();
$requestContext->fromRequest($request); // el contexto se actualiza con la info del request

$locator = new FileLocator(array(APP_PATH . 'config/')); // nos ubicamos en la carpeta app/config
$loader = new YamlFileLoader($locator); //creamos el loader

$options = array(
    'cache_dir' => APP_PATH . 'cache/', //directorio donde serán cacheadas las rutas
    'debug' => DEBUG, // depende de la constante creada en public/web.php
);

$router = new Router($loader, 'routing.yml', $options, $requestContext);

//configuración de ruteo

try {
    $match = $router->match($request->getPathInfo()); //obtenemos la uri desde el pathinfo
    $request->attributes->add($match); //agregamos los datos definidos para la ruta en el request
    //leemos el indice _file definido en la opción defaults de la ruta.
    $file = APP_PATH . 'controller/' . $match['_file'];

    if (!is_file($file)) {
        throw new InvalidArgumentException("No existe el archivo controlador " . $file);
    }

    ob_start(); //vamos a capturar la salida para agregarla luego a un objeto response
    $response = require $file;
    /* Los controladores pueden devolver instancias de Response 
     * o imprimir su contenido Directamente.
     *
     * Si no se devuelve una instancia de response, la creamos y le pasamos el contenido del buffer
     */
    if (!($response instanceof Response)) {
        $response = new Response(ob_get_clean());
    }
} catch (ResourceNotFoundException $e) {
    if (DEBUG) {
        throw new ResourceNotFoundException("No existe una definición para la url "
        . $request->getPathinfo(), $e->getCode(), $e); //en desarrollo mostramos la excepción
    } else {
        // en producción creamos una respuesta
        $response = new Response('Internal Server Error', 500);
    }
}

return $response; //devolvemos la respuesta
{% endhighlight %}

Ahora incluimos lo siguiente en el archivo en **public/web.php**:
{% highlight php linenos %}
<?php // proyecto/public/web.php

require_once __DIR__ . '/../vendor/autoload.php';

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Debug\Debug;

Debug::enable();
//Debug::enable(null, false); //descomentar en producción y comentar el anterior

define('APP_PATH', dirname(__DIR__) . '/app/'); //contiene la ruta hasta app
define('DEBUG', true); // true en desarollo y false en producción

$request = Request::createFromGlobals(); //creamos el objeto Request

$response = require_once APP_PATH . 'bootstrap.php'; //cargamos el bootstrap

$response->send(); //enviamos la respuesta de vuelta
{% endhighlight %}

Con esto ya tenemos el routing trabajando, ahora vamos a crear unas rutas y unos controladores para verificar como funciona lo que hemos hecho:

## Creando las Rutas del proyecto

{% highlight yaml linenos %}
# proyecto/app/config/routing.yml
inicio:
    path: /
    defaults:
        _file: inicio.php

# incluyendo otro yml
_paises:
    prefix: /paises
    resource: routing/paises.yml
{% endhighlight %}

Tenemos una ruta **inicio** y hemos incluido un yml para las rutas que corresponden al manejo de paises en la app.

Es importante destacar, que una ruta puede además de definir un path, incluir otro archivo de rutas, así se logra mantener una mejor organizacion de las mismas mediante archivos más pequeños (y esto no desmejora el rendimiento porque todo luego se cachea en un unico archivo :-D), y se pueden prefijar esas rutas con patrones de url, en este caso el prefijo /paises.

El archivo **app/config/routing/paises.yml**:

{% highlight yaml linenos %}
# proyecto/app/config/routing/paises.yml

# Todas estas rutas estarán automáticamente prefijadas con el patron /paises
# si más adelante lo queremos cambiar, solo debemos hacerlo una ves en el archivo routing.yml :-)

paises_listado:
    path: /
    defaults:
        _file: paises/lista.php  # Archivo app/controller/paises/lista.php

paises_crear:
    path: /crear
    defaults:
        _file: paises/crear.php

paises_editar:
    path: /editar/{nombre}
    defaults:
        _file: paises/editar.php

paises_listado_json:
    path: /all.json
    defaults:
        _file: paises/lista.json.php  # Archivo app/controller/paises/lista.php
{% endhighlight %}

Ahora crearemos algunos de los controladores, para dar un ejemplo de como funciona la app:

Archivos de la ruta inicio:
{% highlight php linenos %}
<?php
# app/controller/inicio.php
include VIEW_PATH . 'inicio.php';
{% endhighlight %}

La Vista:
{% highlight php linenos %}
# app/views/inicio.php
<html>
    <head>
        <title>Título de la Aplicación</title>
    </head>
    <body>
        <h1>Inicio de la Aplicación</h1>        
        <a href="<?php echo $router->generate('paises_listado_json') ?>">Listado de Paises</a>        
    </body>
</html>
{% endhighlight %}

Archivos de la ruta paises_listado:
{% highlight php linenos %}
<?php
# app/controller/paises/lista.php
include VIEW_PATH . 'paises/lista.php';
{% endhighlight %}

La Vista:
{% highlight php linenos %}
# app/views/inicio.php
<html>
    <head>
        <title>Título de la Aplicación</title>
    </head>
    <body>
        <h1>Lista de Paises</h1>        
        <a href="<?php echo $router->generate('paises_editar', array('nombre' => 'Venezuela')) ?>">
            Venezuela
        </a><br/>
        
        <a href="<?php echo $router->generate('paises_editar', array('nombre' => 'Colombia')) ?>">
            Colombia
        </a><br/>
        
        <a href="<?php echo $router->generate('paises_listado_json') ?>">
            Todos en JSON
        </a><br/>        
    </body>
</html>
{% endhighlight %}

Archivo de la ruta paises_editar:
{% highlight php linenos %}
<?php
# app/controller/paises/editar.php
<?php

use Symfony\Component\HttpFoundation\Response;

//leemos el id desde el request

$pais = $request->get('nombre');

return new Response("Editando el pais " . $pais); //devolvemos directamente una instancia de Response
{% endhighlight %}

Archivo de la ruta paises_listado_json:
{% highlight php linenos %}
<?php
# app/controller/paises/lista.xml.php
<?php

use Symfony\Component\HttpFoundation\JsonResponse;

$paises = array(
    'Venezuela', 'Colombia',
);

return new JsonResponse($paises);
{% endhighlight %}

Con esto ya tenemos funcionando una aplicación con el componente de rutas de symfony :-)

Más información por acá [El componente Routing](http://gitnacho.github.io/symfony-docs-es/components/routing/introduction.html)
