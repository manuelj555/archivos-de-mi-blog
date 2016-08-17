---
layout: post
title: El componente DependencyInjection (Extensiones y Compilers)
---

Esta es una continuación del post sobre el uso del [Componente DependencyInjection]({% post_url 2014-02-27-inyeccion-de-dependencias %}), y se hablará sobre la creación de extensiones, que no son más que clases que nos permiten agregar y modificar servicios y parametros al contenedor.

Una información detallada sobre las extensiones se puede encontrar acá:

*   [Gestionando la configuración con extensiones](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/compilation.html#gestionando-la-configuracion-con-extensiones)
*   [Managing Configuration with Extensions](http://symfony.com/doc/current/components/dependency_injection/compilation.html#managing-configuration-with-extensions)

Aparte de las extensiones, el componente permite trabajar con **Compiler Passes** que son clases que alteran los servicios al realizar el proceso de compilación (comprueban la validez del contenedor, optimizan la configuración, quitan los servicios privados y abstractos, resuelven los alias, etc)

Una información detallada sobre los Compiler Passes se puede encontrar acá:

*   [Compilando el contenedor](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/compilation.html)
*   [Crear un CompilerPass](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/tags.html#crear-un-compilerpass)
*   [Creando un pase del compilador](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/compilation.html#creando-un-pase-del-compilador)
*   [Registrando un pase del compilador](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/compilation.html#registrando-un-pase-del-compilador)

## Usando las Extensiones

Para hacer un uso práctico de las extensiones, crearemos el archivo **app/container_configuration.php** en el cual registraremos las extensiones y los Compilers de nuestra app:

{% highlight php linenos %}
<?php

return array(
    'extensions' => array(),//acá vamos a ir añadiendo las instancias de las extensiones
    'compilers' => array(), //y acá las instancias de los Compiler Passes
);

{% endhighlight %}

Ahora vamos a modificar el archivo **app/bootstrap.php** para que quede de la siguiente manera:

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

    $config = require_once APP_PATH . 'container_configuration.php';

    foreach($config['extensions'] as $extension){
        $containerBuilder->registerExtension($extension);//registramos las extensiones en el container
    }

    foreach($config['compilers'] as $compiler){
        $containerBuilder->addCompilerPass($compiler);//registramos los compilers en el container
    }

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
{% endhighlight %}

Como se puede ver se agregaron dos **foreach** que registran las extensiones y compilers al container, además, se han quitado los seteos de los parametros **app_path** y **debug**, pues estos serán establecidos mediante extensiones.

Procedemos ahora a crear nuestra primera extensión, pero antes vamos a añadir un directorio al autoloader para que busque nuestras clases allí, en el composer.json añadimos:

{% highlight json linenos %}
{
    "require": {
        "symfony/http-foundation": "*",
        "symfony/dependency-injection": "*",
        "symfony/yaml": "*",
        "symfony/config": "*"
    },
    "autoload": {
        "psr-0": {
            "": ["app/src/"]
        }
    }
}
{% endhighlight %}

Luego ejecutamos el comando:
    
    php composer dump-autoload

## Creando la Extensión

Ahora creamos la primera extensión en **app/src/Application/Extension/ApplicationExtension.php**:

{% highlight php linenos %}
<?php //   app/src/Application/Extension/ApplicationExtension.php

namespace Application\Extension;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\Extension;

/**
 * @author Manuel Aguirre <programador.manuel@gmail.com>
 */
class ApplicationExtension extends Extension
{

    protected $appPath;
    protected $isDebug;

    public function __construct($appPath, $isDebug)
    {
        $this->appPath = $appPath;
        $this->isDebug = $isDebug;
    }

    public function load(array $config, ContainerBuilder $container)
    {
        //ahora es en la extensión donde establecemos los parametros
        $container->setParameter('app_path', $this->appPath);
        $container->setParameter('debug', $this->isDebug);
    }

}
{% endhighlight %}

Y agregamos la extension al **container_configuration.php**:

{% highlight php linenos %}
<?php

use Application\Extension\ApplicationExtension;

return array(
    'extensions' => array(
        new ApplicationExtension(APP_PATH, DEBUG),
    ),
    'compilers' => array(),
);
{% endhighlight %}

Ya hemos registrado nuestra primera extensión al container, sin embargo nuestra extensión no será cargada, esto es debido a que por defecto el componente utiliza un CompilerPass ([MergeExtensionConfigurationPass](https://github.com/symfony/DependencyInjection/blob/master/Compiler/MergeExtensionConfigurationPass.php#L40)) que solo carga las extensiones que poseen configuración definida en los archivos de configuración (YML, PHP, XML, ...), y en nuestro caso no hemos definido ninguna configuración.

Una explicación sobre definiciones de configuración para extensiones se puede ver en: [Gestionando la configuración con extensiones](http://gitnacho.github.io/symfony-docs-es/components/dependency_injection/compilation.html#gestionando-la-configuracion-con-extensiones)

Para solventar este inconveniente crearemos un CompilerPass que carge las extensiones que no poseen configuración, tal cual lo hace el componente [HttpKernel de Symfony](https://github.com/symfony/HttpKernel/blob/master/DependencyInjection/MergeExtensionConfigurationPass.php#L34)

Extenderemos de la clase **MergeExtensionConfigurationPass** y sobreescribiremos el método **process**, luego le diremos al contenedor que el compiler que cargará las extensiones será el nuestro. El código del compiler es el siguiente:

{% highlight php linenos %}
<?php //   app/src/Application/Compiler/MergeExtensionsPass.php

namespace Application\Compiler;

use Symfony\Component\DependencyInjection\Compiler\MergeExtensionConfigurationPass;
use Symfony\Component\DependencyInjection\ContainerBuilder;

/**
 * @author Manuel Aguirre <programador.manuel@gmail.com>
 */
class MergeExtensionsPass extends MergeExtensionConfigurationPass
{

    public function process(ContainerBuilder $container)
    {
        foreach ($container->getExtensions() as $name => $extension) {
            if (!count($container->getExtensionConfig($name))) {
                //cargamos solo las extensiones que no poseen configuración, 
                //ya que la clase padre cargará las que si poseen config.
                $container->loadFromExtension($name, array());
            }
        }

        parent::process($container);
    }

}
{% endhighlight %}

Este compiler podemos agregarlo al archivo container_configuration.php, pero para hacer mejor las cosas, lo estableceremos directamente en el contenedor de la siguiente manera:

{% highlight php linenos %}
<?php

$containerBuilder->getCompiler()
            ->getPassConfig()->setMergePass(new MergeExtensionsPass());

// Obtenemos el compilador, el cual posee un objeto PassConfig que 
// contiene los compilers que por defecto usa el container, estos podemos
// cambiarlos llamando a algunos métodos como el que hemos usado en este
// caso particular (setMergePass)
// Con esto hacemos que el compiler use nuestra clase para cargar las extensiones.

{% endhighlight %}

Dicha linea de código la debemos agregar al archivo **app/bootsrap.php**, quedando su código de la siguiente manera:

{% highlight php linenos %}
<?php

use Application\Compiler\MergeExtensionsPass;
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

    $config = require_once APP_PATH . 'container_configuration.php';

    foreach ($config['extensions'] as $extension) {
        $containerBuilder->registerExtension($extension); //registramos las extensiones en el container
    }

    foreach ($config['compilers'] as $compiler) {
        $containerBuilder->addCompilerPass($compiler); //registramos los compilers en el container
    }

    $containerBuilder->getCompiler()
            ->getPassConfig()->setMergePass(new MergeExtensionsPass());

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
{% endhighlight %}

Con esto ya podemos registrar nuestras propias extensiones al contenedor, Más adelante veremos como hacer un uso más avanzado de las extensiones y compilers, usando el componente EventDispatcher y el Routing de Symfony.
