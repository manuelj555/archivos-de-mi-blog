---
layout: post
title: Como me ha ayudado el Componente DependencyInjection de Symfony
---

Recientemente he estado trabajando en una aplicación que ya tiene un buen tiempo desarrollada, la labor de nuestro equipo de trabajo en este proyecto es implementar un rediseño de la plataforma, y mejorar/ajustar algunas funcionalidades.

Debido a que la aplicación fue desarrollada hace unos años ya, algunas de las metodologías usadas no utilizan las buenas practicas de desarrollo que podemos encontrar hoy en día.

Me tomé la iniciativa de incorporar un contenedor de servicios (El de Symfony fué mi opción), ya que necesitabamos incorporar algunas libs y clases en la app (como Twig, Router, HttpFoundation, etc...), además, en la plataforma existen varias instancias "globales" de ciertas clases que son de utilidad y son usadas en la mayoria de páginas (manejo de sesión, helpers para formularios, envio de correo, entre otros...).

Esto conlleva al inconveniente de tener instancias en cada petición, aunque no vayan a ser utilizadas en ningún momento, lo que genera un uso de memoria muchas veces innecesario.

Otro aspecto de esto, es que no tenemos la posibilidad de extender dichas implementaciones, para por ejemplo, cambiar su funcionalidad en desarrollo (Agregar Logs de consultas, de emails, inhabilitar el envio de correos, etc).

Y por ultimo pero no menos importante, es que se debe tener mucho cuidado de no sobreescribir esas variables, ya que pueden dañar la funcionalidad de alguna parte de la aplicación.

Acá tenemos un ejemplo de lo que me refería anteriormente:

### Ejemplo del Código Heredado

{% highlight php linenos %}
<?php

$emailGlobal = new GlobalEmail(EMAIL_USER, EMAIL_PASS, EMAIL_HOST);
$formHelperGlobal = new GlobalFormHelper(BASE_PATH);
$sessionGlobal = GlobalSession('namespace');

{% endhighlight %}

La idea era incorporar estas instancias en el contenedor de symfony, para que sea el quien se encargue de la creación de los objetos.

### Instalando el contenedor

Lo primero fué instalar y configurar el componente:

*   [Componente DependencyInjection 1]({% post_url 2014-02-27-inyeccion-de-dependencias %})
*   [Componente DependencyInjection 2]({% post_url 2014-03-02-inyeccion-de-dependencias-extensiones %})

### Creando los servicios

Luego creamos las definiciones de las clases como servicios en el contenedor:

{% highlight yaml linenos %}
//servicios.yml
parameters:
    session_namespace: mi-namespace-de-session

services:
    session:
        class: GlobalSession
        arguments [%session_namespace%]

    form_helper:
        class: GlobalFormHelper
        arguments 
            - @= service('request_stack').getMasterRequest().getBasePath()
{% endhighlight %}

Como ven, es muy facil crear las definciones para nuestras clases/servicios.

Lo siguiente fué cambiar el código de la parte que creaba las instancias de estas clases:

### Cambiando el código

{% highlight php linenos %}
<?php

//la instancia de email llevó un poco más de trabajo.

$formHelperGlobal = $container->get('form_helper');
$sessionGlobal = $container->get('session');

//esto aun no soluciona el problema de tener las instancias así no se usen, 
//pero es un primer paso para dejar de usar las variables globales
//y obtenerlas directo del contenedor.

{% endhighlight %}

Para el servicio email, se optó por crear una clase que sirviera de Factory, ya que GlobalEmail utilizaba parametros que no podian ser registrados en el contenedor.

### EmailFactory al rescate 

{% highlight php linenos %}
<?php

class EmailFactory
{
    public static function createInstance()
    {
        //acá la lógica para obtener los parametros.
        return new GlobalEmail(EMAIL_USER, EMAIL_PASS, EMAIL_HOST);
    }
}

{% endhighlight %}

Luego registramos el servicio:

{% highlight yaml linenos %}
//servicios.yml

services:
    ...

    email:
        class: GLobalEmail
        factory_class: EmailFactory
        factory_method: createInstance

{% endhighlight %}

Como se puede ver el contenedor de servicios es muy flexible, y siempre existe la manera de convertir una clase "global" en un servicio del contenedor. 

Tambien se pudo registrar el servicio usando php, pero al cachearse el contenedor, los parametros que le llegan a email, tambien se cachearán (algo que no podia pasar en nuestro caso).

## Logueando los Correos Enviados

Algo que nos ha permitido la utilización del contenedor es extender las funcionalidades de algunas clases y libs, una de esas funcionalidades ha sido agregar un log de los correos.


{% highlight php linenos %}
<?php

class DebugEmail extends GlobalEmail
{
    protected $realEmail;
    protected $logger;

    public function __construct(GlobalEmail $email, LoggerInterface $logger = null)
    {
        $this->realEmail = $email;//nuestra propiedad es la que realmente envia los correos
        $this->logger = $logger;
    }

    /**
     * Reescribimos el método que envia en correo, para añadir el log.
     */
    public function enviar($from, $to, $subject, $content)
    {
        $this->logger->log("Se ha enviando un email a {$to}, asunto: {$subject}");

        //acá dejamos que la instancia real de email haga el envio normalmente.

        $this->realEmail->enviar($from, $to, $subject, $content);
    }
}

{% endhighlight %}

Hemos creado una clase que extiende de GlobalEmail para así (Implementando el patron Decorator) poder realizar tareas extras sin cambiar el código de la clase original.

### Cambiando la instancia devuelta por el contenedor.

El contenedor de symfony es tan flexible que permite cambiar la construcción de nuestros servicios en alguna de las etapas de compilación del mismo, lo podemos hacer en los [CompilerPass]({% post_url 2014-03-02-inyeccion-de-dependencias-extensiones %}) o en las [Extensiones]({% post_url 2014-03-02-inyeccion-de-dependencias-extensiones %}).

En nuestro caso optamos por hacerlo en una extensión, los pasos a seguir fueron los siguientes:

creamos el servicio debug_email:

{% highlight yaml linenos %}
//servicios.yml

services:
    ...

    debug_email:
        class: DebugEmail
        arguments:
            - @email_real # acá debe ir la instancia de la clase Email real.
            - @?logger  # para hacer los logs (servicio definido en alguna parte)

# Aunque el servicio email_real no exista, nosotros le haremos llegar el
# mismo al servicio debug_email desde la extension.

{% endhighlight %}

Ahora en nuestra extensión, le pasamos la instancia de GlobalEmail al servicio en su primer argumento, y además le decimos al contenedor, que cuando se pida el servicio **email** devuelva realmente **debug_email**

{% highlight php linenos %}
<?php

//dentro del metodo load de nuestra extensión registrada
//y luego de condicionar que estemos en desarrollo:

$emailDefinition = $container->findDefinition('email');
$debugEmailDefinition = $container->findDefinition('debug_email');

$emailDefinition->setPublic(false); //hacemos invisible el servicio real
$container->setDefinition('email_real', $emailDefinition);//creamos el servicio email_real a partir de email.
$container->setAlias('email', 'debug_email'); //y le decimos al contenedor que cuando pidan el servicio
//email, devuelva debug_email

{% endhighlight %}

Con el código anterior (Que puede parecer muy confuso, nosotros lo hemos venido viendo en el core del framework syfmony (-: ) logramos que el servicio email, devuelva la instancia de DebugEmail en vez de GlobalEmail.

Acá podemos ver un ejemplo de lo que se ha cacheado:

{% highlight php linenos %}
<?php
//clase del contenedor cacheado
/**
 * Este código es autogenerado por el componente, en base a las definciones.
 *
 * Se llamará a este metodo al solicitar a debug_email service.
 */
protected function getDebugEmailService()
{
    $emailReal = call_user_func(array('EmailFactory', 'createInstance'); //llamamos al factory
    $logger = $this->get('logger');

    return $this->services['debug_email'] = new DebugEmail($emailReal, $logger);
}

{% endhighlight %}

La variable $emailReal se crea dentro del método **getDebugEmailService** debido a que le hemos dicho que no es publica (**$emailDefinition->setPublic(false)**).

Aparte, en algun punto del contenedor cacheado hay un arreglo de aliases, y uno de sus valores es el alias de email hacia debug__email, por lo que cuando se pida el servicio email, primero se buscará en los alias, y así se devolverá realmente debug__email.

### Conclusión

Para mi el utilizar un contenedor (y más aun el de symfony) es de mucha ayuda para mejorar y organizar el código de nuestras aplicaciones.

