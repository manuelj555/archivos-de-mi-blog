---
layout: post
title: Creando un Foro con el MicroKernelTrait de Symfony (Parte 3)
---

Esta es la tercera parte de una serie de artículos donde se explicará paso a paso el desarrollo de una aplicación muy simple utilizando el [MicroKernelTrait de Symfony](http://symfony.com/doc/current/configuration/micro_kernel_trait.html).

[Parte 1](/2016/08/14/foro-micro-kernel-trait-symfony-parte-1.html) [Parte 2](/2016/08/21/foro-micro-kernel-trait-symfony-parte-2.html)

_**Nota**: Los posts se enfocarán a usuarios que tengan conocimientos previos de Symfony, que conozcan sobre el AppKernel, los directorios básicos de un proyecto symfony y de los archivos de configuración y rutas._

- - -

En esta parte vamos a trabajar en dos cosas:

 * Implementación de seguridad con el bundle SecurityBundle.
 * Implementación del DebugBundle para depuración mientras desarrollamos.
 * Implementación del WebProfilerBundle para depuración mientras desarrollamos.

# Agregando seguridad en la aplicación

En esta parte se va a trabajar con la seguridad de la aplicación. Ahora los usuarios deberán iniciar sesión en la plataforma para poder crear preguntas.

La estructura de archivos que se va a crear/editar en esta parte es la siguiente:

	app/config/config.yml                                      // Edición
    app/config/security.yml                                    // Creación
    app/AppKernel.php                                          // Edición
    app/Resources/views/base.html.twig                         // Edición
    src/App/Controller/QuestionController.php                  // Edición
	web/css/foro.css                                           // Edición

- - -

### Agregando los Bundles

Lo primero es registrar los bundles en el `registerBundles` del `AppKernel.php` (Lineas 6, 9-14):

```php
<?php
//... Método registerBundles
$bundles = array(
    //...
    // se añade el SecurityBundle
    new Symfony\Bundle\SecurityBundle\SecurityBundle(),
);

if (in_array($this->getEnvironment(), array('dev', 'test'))) {
    // ...
    $bundles[] = new Symfony\Bundle\DebugBundle\DebugBundle();
    // se añade el WebProfilerBundle
    $bundles[] = new Symfony\Bundle\WebProfilerBundle\WebProfilerBundle();
}
// ...
```

Seguidamente agregamos la configuración del `WebProfilerBundle` en el método `configureContainer` (Lineas 8-13):

```php
<?php
// ... 
protected function configureContainer(ContainerBuilder $c, LoaderInterface $loader)
{
    $loader->load(__DIR__.'/config/config.yml');

    // Se añade la config del WebProfilerBundle, solo si el bundle fué cargado.
    if (isset($this->bundles['WebProfilerBundle'])) {
        $c->loadFromExtension('web_profiler', array(
            'toolbar' => true,
            'intercept_redirects' => false,
        ));
    }
}
```

Para finalizar importamos las rutas desde el método `configureRoutes` (Lineas 6-9):

```php
<?php
// ... 
protected function configureRoutes(RouteCollectionBuilder $routes)
{
    // Importamos las rutas del WebProfilerBundle, solo si el bundle fué cargado.
    if (isset($this->bundles['WebProfilerBundle'])) {
        $routes->import('@WebProfilerBundle/Resources/config/routing/wdt.xml', '/_wdt');
        $routes->import('@WebProfilerBundle/Resources/config/routing/profiler.xml', '/_profiler');
    }

    // Cargamos los controllers usando anotaciones
    $routes->import(__DIR__.'/../src/App/Controller/', '/', 'annotation');
}
```

### Ajustando el config.yml

En el `config.yml` se va a importar el recurso `app/config/security.yml` (linea 3):

```yaml
imports:
    - { resource: parameters.yml }
    - { resource: security.yml } # Se importa el app/config/security.yml
    - { resource: services.yml }

framework:
    secret: StringAleatorio
    templating:
        engines: ['twig']
    profiler: { only_exceptions: false }
    form: ~                       # Nos permite crear y trabajar con formularios de Symfony.
    validation:                   # Activa la validación de objetos.
        enable_annotations: true
    translator:                   # Activa la traducción de las validaciones.
        fallback: es
    default_locale: es

# Registramos un tema de formularios para mejorar el aspecto de los mismos.
twig:
    form_theme:
        - 'form/fields.html.twig'
```

Por último vamos a crear el fichero `app/config/security.yml` para configurar el bundle de seguridad:

### El security.yml

```yaml
# app/config/security.yml
security:
    # Para efectos del ejemplo las claves no se encriptan
    encoders:
        Symfony\Component\Security\Core\User\User: plaintext

    providers:
        # Usamos el proveedor de usuarios en memoria para no complicar el tutorial:
        in_memory:
            memory:
                users:
                    one@test.com:
                        password: 123
                        roles: 'ROLE_USER'
                    two@test.com:
                        password: 123
                        roles: 'ROLE_USER'
                    three@test.com:
                        password: 123
                        roles: 'ROLE_USER'

    firewalls:
        # Las rutas del profiler no llevan seguridad.
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false

        # Creamos el firewall de toda la aplicación:
        main:
            pattern: ^/
            anonymous: ~
            # La autenticación la manejamos con http_basic para simplificar el tutorial
            http_basic: ~
```

Con estos ajustes ya tenemos el `WebProfilerBundle`, el `DebugBundle` y el `SecurityBundle` debidamente configurados en la aplicación.

#### Restringiendo el acceso al crear preguntas

Ahora debemos modificar el método `QuestionController::newAction` para que solo puedan crear preguntas los usuarios que hayan iniciado sesión en la plataforma. Para ello vamos a usar la anotación `@Security("is_authenticated()")` en el método que crea preguntas, y además vamos a ajustar su lógica para relacionar la pregunta al usuario logueado:

```php
<?php
// ... 
// Importante añadir el use de la anotación Security: 
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Security;
// ...

/**
 * @Route("/new", name="question_create")
 * @Security("is_authenticated()")  Se añade la anotación de seguridad
 */
public function newAction(Request $request)
{
    $form = $this->createForm(QuestionType::class, null, [
        'author' => $this->getUser()->getUsername(), // Se pasa el usuario logueado al form.
    ]);
    $form->handleRequest($request);

    if ($form->isSubmitted() and $form->isValid()) {
        $this->get('repository.question')->save($form->getData());

        return $this->redirectToRoute('question_list');
    }

    return $this->render('question/new.html.twig', [
        'form' => $form->createView(),
    ]);
}
```

El código anterior añade seguridad a la acción `newAction` (linea 9), y relaciona la pregunta con el usuario logueado (linea 14). Recordemos que en capítulos anteriores se creó un Formulario `QuestionType` y en él se definió una opción `author` que se pasa al objeto `Question` al crearlo.

### Mostrando la información del usuario

Vamos a ajustar el archivo `app/Resources/views/base.html.twig` para que aparesca la información del usuario logueado en el header de la página:

{% raw %}
```twig
{# app/Resources/views/base.html.twig #}
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
            {% if app.user and is_granted('IS_AUTHENTICATED_FULLY') %}
                <div class="header-user-container">
                    <span class="header-user">
                        {{ app.user.username }}                        
                    </span>
                </div>
            {% endif %}
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

Se añaden las lineas **16 a la 22**, donde se muestra el nombre del usuario (su correo para efectos del tutorial) a la derecha del título de la aplicación.

Para visualizar correctamente los estilos con los cambios añadidos recientemente se debe actualizar el archivo `web/css/foro.css` con el siguiente contenido:

```css
/*! normalize.css v4.1.1 | MIT License | github.com/necolas/normalize.css */html{font-family:sans-serif;line-height:1.15;-ms-text-size-adjust:100%;-webkit-text-size-adjust:100%}body{margin:0}article,aside,details,figcaption,figure,footer,header,main,menu,nav,section,summary{display:block}audio,canvas,progress,video{display:inline-block}audio:not([controls]){display:none;height:0}progress{vertical-align:baseline}template,[hidden]{display:none}a{background-color:transparent;-webkit-text-decoration-skip:objects}a:active,a:hover{outline-width:0}abbr[title]{border-bottom:0;text-decoration:underline;text-decoration:underline dotted}b,strong{font-weight:inherit}b,strong{font-weight:bolder}dfn{font-style:italic}h1{font-size:2em;margin:.67em 0}mark{background-color:#ff0;color:#000}small{font-size:80%}sub,sup{font-size:75%;line-height:0;position:relative;vertical-align:baseline}sub{bottom:-0.25em}sup{top:-0.5em}img{border-style:none}svg:not(:root){overflow:hidden}code,kbd,pre,samp{font-family:monospace,monospace;font-size:1em}figure{margin:1em 40px}hr{box-sizing:content-box;height:0;overflow:visible}button,input,optgroup,select,textarea{font:inherit;margin:0}optgroup{font-weight:bold}button,input{overflow:visible}button,select{text-transform:none}button,html [type="button"],[type="reset"],[type="submit"]{-webkit-appearance:button}button::-moz-focus-inner,[type="button"]::-moz-focus-inner,[type="reset"]::-moz-focus-inner,[type="submit"]::-moz-focus-inner{border-style:none;padding:0}button:-moz-focusring,[type="button"]:-moz-focusring,[type="reset"]:-moz-focusring,[type="submit"]:-moz-focusring{outline:1px dotted ButtonText}fieldset{border:1px solid silver;margin:0 2px;padding:.35em .625em .75em}legend{box-sizing:border-box;color:inherit;display:table;max-width:100%;padding:0;white-space:normal}textarea{overflow:auto}[type="checkbox"],[type="radio"]{box-sizing:border-box;padding:0}[type="number"]::-webkit-inner-spin-button,[type="number"]::-webkit-outer-spin-button{height:auto}[type="search"]{-webkit-appearance:textfield;outline-offset:-2px}[type="search"]::-webkit-search-cancel-button,[type="search"]::-webkit-search-decoration{-webkit-appearance:none}::-webkit-input-placeholder{color:inherit;opacity:.54}::-webkit-file-upload-button{-webkit-appearance:button;font:inherit}*{box-sizing:border-box;font-family:sans-serif}body{font-size:14px}header{padding:30px 0;margin:0;background-color:#82b440;box-shadow:5px 10px 10px #555}header .header-title{color:white;font-size:3em}header .header-title .header-user-container{float:right}header .header-title .header-user-container .header-user{font-size:.5em;vertical-align:middle}.container{margin:0 auto;padding:0 10px;width:800px}.container .page-header{border-bottom:1px solid #ccc;padding:20px 0;font-size:2.4em;margin-bottom:40px}.container .page-header .button{font-size:.48em;vertical-align:middle}.question{margin-bottom:30px;border-bottom:1px solid #ccc}.question h3{font-size:1.2em;margin:6px 0}.question h3 a{text-decoration:none;color:#4e6e2f}.question .question-content{text-indent:20px;text-align:justify;margin-bottom:10px}.question .question-footer{font-size:.9em;color:#75a43d;margin-bottom:6px;font-style:italic}.question:last-child{border-bottom:0}.pull-right{float:right}.button,input[type="button"],input[type="submit"],input[type="reset"]{padding:8px;cursor:pointer;text-decoration:none;color:#222;border:1px solid #aaa;border-radius:4px;box-shadow:2px 2px 2px #aaa;display:inline-block;font-size:1.1em}.button.primary,input[type="button"].primary,input[type="submit"].primary,input[type="reset"].primary{color:white;background-color:#82b440;border-color:#5e7f30}.button.primary:hover,input[type="button"].primary:hover,input[type="submit"].primary:hover,input[type="reset"].primary:hover{background:#75a43d}.button:hover,input[type="button"]:hover,input[type="submit"]:hover,input[type="reset"]:hover{background:#f1f1f1}.button:active:hover,input[type="button"]:active:hover,input[type="submit"]:active:hover,input[type="reset"]:active:hover{transform:translateX(1px) translateY(1px)}input[type="submit"]{color:white;background-color:#82b440;border-color:#5e7f30}input[type="submit"]:hover{background:#75a43d}form .form-row{margin-bottom:20px}form .form-row:last-child{margin-bottom:30px}form label{display:block;font-weight:bold;margin-bottom:4px}form .form-widget{width:100%;padding:4px;line-height:25px;border-radius:4px;border:2px solid #5e7f30;box-shadow:2px 2px 4px #aaa}form .form-widget:focus{box-shadow:2px 2px 8px #555}form .description-widget{min-height:200px}form .has-error label{color:#ae0000}form .has-error .form-widget{border-color:#ae0000}form .form-errors{color:#ae0000}
```

Con estos ajustes si iniciamos el servidor con el comando `php -S localhost:8000 -t web` y accedemos desde un navegador a `http://localhost:8000/` y desde allí presionamos el botón <img src="{{ site.url }}/assets/images/micro-foro/2-boton-hacer-pregunta.png" class="img-inline" alt="Hacer una Pregunta" style="max-width: 130px;" /> vamos a tener una pantalla como la siguiente:

![Login de Usuario]({{ site.url }}/assets/images/micro-foro/3-login-form.png)

Allí nos aparece un cuadro de dialogo donde debemos ingresar usuario y contraseña. Si por ejemplo añadimos en el _Nombre de Usuario_ el valor **one@test.com** y de _contraseña_ ingresamos **123** y presionamos _Aceptar_, vamos a visualizar la página de creación de preguntas con la información del usuario en el header:

![Info Usuario]({{ site.url }}/assets/images/micro-foro/3-crear-pregunta-usuario-logueado.png)

_Nota: Los datos de acceso del usuario ingresados en el cuadro de dialogo están definidos en el `app/config/security.yml` en la configuración de los usuarios en memoria._

En la imagen anterior, en la parte derecha del header de la página, aparece el correo del usuario que inició sesión. Además en la parte inferior se puede visualizar la barra del web profiler, y allí en la pestaña de seguridad aparece tambien el correo del usuario.

Si creamos una pregunta podemos ahora vizualizar la información del usuario que la creó desde el listado de preguntas:

![Info Usuario - Listado]({{ site.url }}/assets/images/micro-foro/3-info-usuario-listado.png)

Como se puede apreciar, ahora en el listado aparecen los datos de los autores de las preguntas.

Bueno, hasta acá la tercera parte de la construcción del micro foro. Más adelante seguiremos con la administración de las preguntas, Permitiendo visualizar una pregunta y agregarle respuestas a dicha pregunta.

[Descargar o visualizar el Proyecto](https://github.com/manuelj555/foro-microkerneltrait/releases/tag/foro-parte-3)