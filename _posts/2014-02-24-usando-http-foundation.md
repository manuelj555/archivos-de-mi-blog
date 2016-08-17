---
layout: post
title: Usando el componente HttpFoundation
---
El componente HttpFoundation de Symfony permite encapsular y manejar la información de una petición de manera orientada a objetos, y ofrece una capa robusta que nos abstrae de tareas que pueden resultar tediosas de hacer a mano.

Basicamente el componente ofrece una serie de clases que permiten acceder a la información de la petición (Request) y clases que facilitan la creación de respuestas http a devolver (Response).

## ¿Porque usar este componente?

En mi opinión este componente facilita la realización de tareas que de otra forma llegan a ser engorrosas y tediosas, ejemplo de ellos son:

*   Subida de archivos al servidor.
*   Manejo de cookies.
*   Gestión de Sessión (Sesión por defecto, en Base de datos, en archivos, etc...).
*   Saber si una petición es asincrona.
*   Verificar que un parametro venga o exista en alguna de las variables super globales.
*   Obtener información de la url, como el puerto, el queryString, pathinfo, host, etc...
*   Seteo de los headers para la respuesta (algo que muchas veces no hacemos).
*   Manejo de Códigos de estado para la respuesta.
*   Creación de headers para descarga de archivos.
*   Entre otras cosas que aun no he usado...

Y lo más importante, si ya existe algo que nos ofrece todo esto y lo hace bien, ¿porque vamos a estar reinventando la rueda haciendo las cosas de cero?.

## Instalando el componente

Para descargar el componente lo recomendable es hacerlo mediante composer, archivo composer.json:
{% highlight json linenos %}
{
    "require": {
        "symfony/http-foundation": "*"
    }
}
{% endhighlight %}

Ahora solo hace falta ejecutar el comando:

    php composer install

## Creando el objeto Request

Crear la instancia del Request es tan sencillo como ejecutar el siguiente código:
{% highlight php linenos %}
<?php //proyecto/index.php

require_once __DIR__ . '/vendor/autoload.php';

use Symfony\Component\HttpFoundation\Request;

$request = Request::createFromGlobals(); //acá creamos el objeto request
{% endhighlight %}

El método **createFromGlobals()** crea una instancia con la información de la petición a partir de las variables globales de php $_GET, $_POST, $_REQUEST, $_SERVER, $_FILES y $_COOKIE.

Ahora podemos obtener la info del request desde los atributos del objeto:

##### El atributo query:

Este atributo es una instancia de **(\Symfony\Component\HttpFoundation\ParameterBag)** y contiene la información contenida en la variable global $_GET, para usarlo desde el objeto request lo hacemos de la siguiente manera:

{% highlight php linenos %}
<?php

echo $request->query->get('nombre'); 
//devuelve el valor contenido en $_GET['nombre'] si existe, si no devuelve null

echo $request->query->get('nombre', 'Manuel'); 
//si no se encuentra el indice nombre en $_GET el método devuelve 'Manuel' por defecto.

echo $request->query->get('usuario[nombre]', 'Manuel', true); 
//acá hay algo interesante, si el indice usuario contiene un array, el componente
//intentará devolver el valor de la siguiente manera: $_GET['usuario']['nombre']
//recordar que para que funcione, se debe pasar true en el tercer argumento.

var_dump($request->query->all());

{% endhighlight %}

##### El atributo request:

Este atributo es una instancia de **(\Symfony\Component\HttpFoundation\ParameterBag)** y contiene la información contenida en la variable global $_POST, para usarlo desde el objeto request lo hacemos de la siguiente manera:

{% highlight php linenos %}
<?php

echo $request->request->get('nombre'); 
//devuelve el valor contenido en $_POST['nombre'] si existe, si no devuelve null

echo $request->request->get('nombre', 'Manuel'); 
//si no se encuentra el indice nombre en $_POST el método devuelve 'Manuel' por defecto.

echo $request->request->get('usuario[nombre]', 'Manuel', true); 
//acá hay algo interesante, si el indice usuario contiene un array, el componente
//intentará devolver el valor de la siguiente manera: $_POST['usuario']['nombre']
//recordar que para que funcione, se debe pasar true en el tercer argumento.

var_dump($request->request->all());

{% endhighlight %}

##### El atributo server:

Este atributo es una instancia de **(\Symfony\Component\HttpFoundation\ServerBag)** y contiene la información contenida en la variable global $_SERVER, para usarlo desde el objeto request lo hacemos de la siguiente manera:

{% highlight php linenos %}
<?php

echo $request->server->get('QUERY_STRING'); 
//devuelve el valor contenido en $_SERVER['QUERY_STRING'] si existe, si no devuelve null

echo $request->server->get('SCRIPT_FILENAME'); 
echo $request->server->get('DOCUMENT_ROOT'); 

var_dump($request->server->all());

{% endhighlight %}

Igualmente se pueden establecer valores por defecto para cuando el indice no exista.

##### La propiedad attributes:

Este atributo es una instancia de **(\Symfony\Component\HttpFoundation\ParameterBag)**, y se utiliza para establecer información referente a la petición en las aplicaciones, por ejemeplo, el framework symfony almacena allí la info concerniente a la ruta que se está ejecutando (el controlador y la acción a llamar, los parametros enviados en la ruta, etc).

Este objeto es muy util para almacenar información dependiendo de los requerimientos de la app.

{% highlight php linenos %}
<?php

$request->attributes->set('page', 1); 
$request->attributes->set('page', 2); 

echo $request->attributes->get('page', 1); 

var_dump($request->attributes->all());

{% endhighlight %}

##### El atributo files:

Este atributo es una instancia de **(\Symfony\Component\HttpFoundation\FileBag)**, y contiene un arreglo de objetos de tipo **(\Symfony\Component\HttpFoundation\File\UploadedFile)**. listos para ser usados.

Este contenedor es muy util a la hora de leer archivos subidos al servidor, porque los objetos que contiene ofrecen una serie de métodos que nos permitirán manejar los archivos facilmente.

{% highlight php linenos %}
<?php

$file = $request->files->get('foto'); 
$movedFile = $file->move('diretorio'/*, 'nombre.jpg' */); 
//guardar el archivo es tan sencillo como llamar al metodo move()

if($file->isValid()){
    echo $file->getExtension();
    echo $file->getMimeType();
    echo $file->getFilename();
    echo $file->getSize();
    echo $file->getBasename();
    
    $movedFile = $file->move('diretorio'/*, 'nombre.jpg' */); 

}else{
    echo $file->getError(); //devuelve el número del error
}

var_dump($request->files->all());

{% endhighlight %}

##### El atributo cookies:

Este atributo es una instancia de **(\Symfony\Component\HttpFoundation\ParameterBag)**, y contiene la información contenida en la variable global $_COOKIE, para usarlo desde el objeto request lo hacemos de la siguiente manera:

{% highlight php linenos %}
<?php

echo $request->cookie->get('parametro');

var_dump($request->cookie->all());
{% endhighlight %}

##### El atributo headers:

Este atributo es una instancia de **(\Symfony\Component\HttpFoundation\HeaderBag)**, y contiene la información referente a cabeceras http de la petición.

{% highlight php linenos %}
<?php

echo $request->headers->get('host');
echo $request->headers->get('content_type');

var_dump($request->headers->all());

{% endhighlight %}

Los indices contenidos en la colección estan normalizados y en minuscula.

## El método Request::get()

Este método es un atajo para obtener un parametro que puede estar en algunas de las propiedades  query, attributes o request, y mantiene ese mismo orden de procedencia en la busqueda.

Es muy útil cuando por ejemplo se usa el routing de symfony y guardamos los parametros de la url en la propiedad atributes del objeto $request, o lo enviamos por get, este método igual encontrará dicho valor.

{% highlight php linenos %}
<?php

echo $request->get('page');
echo $request->get('page', 1);
echo $request->get('route');

{% endhighlight %}

Supongamos que estamos creando una web sin sistema de ruta pero usando el componente HttpFoundation, y estamos paginando una consulta:

{% highlight php linenos %}
<?php

// URL: http://example.com/paises/?page=3

$currentPage = $request->get('page', 1); //encontrará el valor en $_GET y devolverá un 3
//Paginado ...
echo $currentPage; //mostrará 3
{% endhighlight %}

Si luego en ese proyecto implementamos un sistema de routing, como por ejemplo el de symfony y guardamos los parametros de la url en attributes:

{% highlight yaml linenos %}
paises:
    path: /paises/{page}
    defaults:
        ...
{% endhighlight %}

Ahora el parametro no se envia por GET, sino que forma parte de la url, pero el routing de symfony, guarda el valor de ese parametro en **$request->attributes**, entonces nuestro cógido sigue funcionando sin tocar nada:

{% highlight php linenos %}
<?php

// URL: http://example.com/paises/3

$currentPage = $request->get('page', 1); //encontrará el valor en attributes de request y devolverá un 3
//Paginado ...
echo $currentPage; //mostrará 3
{% endhighlight %}

Este es un buen ejemplo de lo util que puede ser el método get() y el uso práctico de la propiedad attributes.

### Metodos útiles de ParameterBag

Algunos métodos de interes:

*   **get($path, $default = null, $deep = false)**: devuelve un valor contenido en el índice pasado.
*   **has($key)**: devuelve true si existe el indice en la colección.
*   **remove($key)**: elimina un valor contenido en el índice especificado.
*   **getAlnum($key, $default = '', $deep = false)**: devuelve el valor filtrado alfanumericamente.
*   **getAlpha($key, $default = '', $deep = false)**: devuelve el valor filtrado alfabeticamente.
*   **getDigits($key, $default = '', $deep = false)**: devuelve solo los digitos.
*   **getInt($key, $default = '', $deep = false)**: devuelve el valor convertido en int.

## Ventajas de usar el objeto Request

El objeto request ofrece una serie de métodos que nos facilitarán la vida al programar.

Un ejemplo de ello puede ser conocer si la petición es mediante ajax o no, en php seria algo como:

{% highlight php linenos %}
<?php

if (isset($_SERVER['HTTP_X_REQUESTED_WITH'])
    AND strtolower($_SERVER['HTTP_X_REQUESTED_WITH']) === 'xmlhttprequest') {
    // AJAX
}
{% endhighlight %}

Mientras que con el componente seria algo como:

{% highlight php linenos %}
<?php

if ($request->isXmlHttpRequest()) {
    // AJAX
}
{% endhighlight %}

Existen otros métodos en la clase como:

*   getLanguages().
*   getCharsets().
*   getBasePath().

Entre otros métodos, que nada más hecharles una mirada interna, se puede apreciar que nos ahorran un montón de trabajo para obtener esos datos.

# La clase Response

Esta clase permite manejar de forma simple el envio del contenido y las cabeceras que serán devueltas al servidor.

Ejemplo de uso:

{% highlight php linenos %}
<?php

use Symfony\Component\HttpFoundation\Response;

$response = new Response("Hola Mundo!!!"); //crea el objeto con las cabeceras
$response->send(); //envia las cabeceras y el contenido

$response = new Response("No existe la Página :-(", 404); //establecemos el codigo de estado de la respuesta
$response->send();

//enviando una respuesta en formato json
$response = new Response('["Mi valor"]', 200, array('Content-Type' => 'application/json')); 
$response->send();
// otra forma de hacerlo
$response = new Response('["Mi valor"]', 200); 
$response->headers->set('Content-Type', 'application/json');
$response->send();

if($response->isOk()){
    //respuesta exitosa
}

if($response->isRedirection()){
    //se está redireccionando
}

if($response->isRedirect('/pais')){
    //se está redireccionando a /pais
}

if($response->isServerError()){
    //error del servidor :-/
}

{% endhighlight %}

El constructor de la clase espera tres parametros:
{% highlight php linenos %}
<?php

/**
 * Constructor.
 *
 * @param string  $content The response content
 * @param integer $status  The response status code
 * @param array   $headers An array of response headers
 *
 * @throws \InvalidArgumentException When the HTTP status code is not valid
 *
 * @api
 */
 public function __construct($content = '', $status = 200, $headers = array())
{% endhighlight %}

Tambien podemos crear la instancia de la siguiente manera:
{% highlight php linenos %}
<?php

use Symfony\Component\HttpFoundation\Response;

$response = Response::create("Hola Mundo!!!");
$response->send();

//asi podemos hacer algo como:

Response::create("Mi contenido")
    ->setStatusCode(200)
    ->send();

{% endhighlight %}

Es importante destacar que es el método **send()** el que realmente hace el envío de las cabeceras y el contenido.

Además el componente ofrece otra serie de clases de respuesta para facilitar aun más las cosas.

#### Clase RedirectResponse

Esta clase crea una respuesta con el contenido y cabeceras correspondientes a un redirección http, acepta los siguientes parametros en su constructor:

{% highlight php linenos %}
<?php

/**
 * Creates a redirect response so that it conforms to the rules 
 * defined for a redirect status code.
 *
 * @param string  $url     The URL to redirect to
 * @param integer $status  The status code (302 by default)
 * @param array   $headers The headers (Location is always set to the given url)
 *
 * @throws \InvalidArgumentException
 *
 * @see http://tools.ietf.org/html/rfc2616#section-10.3
 *
 * @api
 */
public function __construct($url, $status = 302, $headers = array())
{% endhighlight %}

Ejemplo de uso:
{% highlight php linenos %}
<?php

use Symfony\Component\HttpFoundation\RedirectResponse;

$response = new RedirectResponse("http://google.com");
$response->send();

$response = new RedirectResponse("/otro_archivo_de_mi_host.html");
$response->send();

RedirectResponse::create("http://google.com")->send();

{% endhighlight %}

Esta clase nos ahorra las tarea de: 

*   Crear el header **Location** que hace la redirección http.
*   Crear el header con el código de estado 302 (que podemos cambiar en el segundo argumento del constructor).
*   Crear el html con la etiqueta &lsaquo;meta http-equiv="refresh" content="1;url=mi-url" /&rsaquo;.

#### Clase JsonResponse

Esta clase crea una respuesta con el contenido y cabeceras para el formato json, acepta los siguientes parametros en su constructor:

{% highlight php linenos %}
<?php

/**
 * Constructor.
 *
 * @param mixed   $data    The response data
 * @param integer $status  The response status code
 * @param array   $headers An array of response headers
 */
 public function __construct($data = null, $status = 200, $headers = array())
{% endhighlight %}

Ejemplo de uso:
{% highlight php linenos %}
<?php

use Symfony\Component\HttpFoundation\JsonResponse;

$datos = array(
    'nombre' => 'Manuel',
    'pais' => 'Venezuela',
);

$response = new JsonResponse($datos);
$response->send();

JsonResponse::create($datos)->send();

{% endhighlight %}

Esta clase nos ahorra las tarea de:
 
*   Crear el header **Content-Type** correspondiente.
*   Convertir a json el arreglo que se pasa como parametro.

#### Clase BinaryFileResponse
#### Clase StreamedResponse