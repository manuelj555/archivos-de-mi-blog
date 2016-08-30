---
layout: post
title: Creando un Foro con el MicroKernelTrait de Symfony (Parte 2)
---

Esta es la segunda parte de una serie de artículos donde se explicará paso a paso el desarrollo de una aplicación muy simple utilizando el [MicroKernelTrait de Symfony](http://symfony.com/doc/current/configuration/micro_kernel_trait.html).

[Parte 1](/2016/08/14/foro-micro-kernel-trait-symfony-parte-1.html)

_**Nota**: Los posts se enfocarán a usuarios que tengan conocimientos previos de Symfony, que conozcan sobre el AppKernel, los directorios básicos de un proyecto symfony y de los archivos de configuración y rutas._

- - -

# Manejando las Preguntas

En esta parte se va a trabajar con la administración de las preguntas, lo cual implica:

 * Creación de la Base de Datos.
 * Creación de Preguntas.
 * Visualización de Preguntas (Listado).

La estructura de archivos que se va a crear/editar en esta parte es la siguiente:

	app/config/config.yml                                              // Edición
	app/config/parameters.yml                                          // Creación
	app/config/services.yml                                            // Creación
    app/Resources/views/base.html.twig                                 // Creación
    app/Resources/views/question/list.html.twig                        // Creación
    app/Resources/views/question/new.html.twig                         // Creación
    src/App/Controller/QuestionController.php                          // Creación
    src/App/Form/QuestionType.php                                      // Edición
    src/Forum/Question/Question.php                                    // Creación
    src/Forum/Question/Factory/QuestionFactory.php                     // Creación
    src/Forum/Question/Repository/QuestionRepository.php               // Creación
    src/Forum/Question/Repository/DbQuestionRepository.php             // Creación

- - -

### La Base de Datos

Lo primero que haremos será crear una base de datos para conectarnos a ella. Para efectos del ejemplo se decidió trabajar con una base de datos Mysql. Llamaremos a la base de datos **micro-foro** y crearemos la siguiente tabla:

```sql
CREATE TABLE `questions` (
    `id` INT (11) NOT NULL AUTO_INCREMENT,
    `title` VARCHAR (255) NOT NULL,
    `description` text NOT NULL,
    `author` VARCHAR (255) NOT NULL,
    `created` datetime NOT NULL,
    `resolved` bit (1) NOT NULL,
     PRIMARY KEY (`id`)
) ENGINE = INNODB
```

Se crea la tabla con esa estructura ya que debemos recordar que los atributos de las preguntas son:

 * id
 * title
 * description
 * author
 * createdAt
 * resolved

Con la tabla creada procedemos a trabajar en los archivos y clases necesarios para administrar preguntas en la aplicación.

- - - 

### La clase _Question_

La idea es modelar la tabla en un objeto php para tener una representación de las preguntas en la aplicación. Para lo cual se crea la clase (Entidad o Modelo) que representa una pregunta. 

```php
<?php // src/Forum/Question/Question.php

namespace Forum\Question;

use Symfony\Component\Validator\Constraints as Assert;

class Question
{
    /** @var int */
    private $id;

    /**
     * @var string
     *
     * @Assert\NotBlank(message="Por favor, indique el título de su pregunta")
     */
    private $title;

    /**
     * @var string
     *
     * @Assert\NotBlank(message="Por favor, describa su pregunta")
     */
    private $description;

    /** @var string */
    private $author;

    /** @var \DateTime */
    private $createdAt;

    /** @var bool */
    private $resolved = false;

    public function __construct($title, $description, $author, \DateTime $createdAt = null)
    {
        $this->title = $title;
        $this->description = $description;
        $this->author = $author;
        $this->createdAt = $createdAt ?: new \DateTime('now');
    }

    public function getId() { return $this->id; }

    public function setId($id) { $this->id = $id; }

    public function getTitle() { return $this->title; }

    public function setTitle($title) { $this->title = $title; }

    public function getDescription() { return $this->description; }

    public function setDescription($description) { $this->description = $description; }

    public function getAuthor() { return $this->author; }

    public function getCreatedAt() { return $this->createdAt; }

    public function isResolved() { return $this->resolved; }

    public function markAsResolved() { $this->resolved = true; }
}
```

La clase posee un atributo por cada propiedad de la pregunta, además se agregan algunas [Restricciones de Validación](http://symfony.com/doc/current/validation.html#constraints), aprovenchando el [Validador de Symfony](http://symfony.com/doc/current/validation.html).

### La clase QuestionRepository

Debido a que necesitamos de alguna manera obtener y persistir las preguntas en la aplicación, vamos hacer uso del [patrón repositorio](http://stackoverflow.com/questions/16176990/proper-repository-pattern-design-in-php) el cual nos permite mapear los modelos contra una fuente de datos que puede ser una base de datos, una api rest, la sesión, etc.

Teniendo en cuenta que debemos poder crear, listar y actualizar preguntas en la base de datos, vamos a definir los siguientes métodos para la interfaz:

 * **save**: Crea o actualiza una pregunta en la base de datos.
 * **find**: Obtiene una pregunta en base a su id.
 * **findAll**: Obtiene las preguntas ordenadas de forma descendiente.

El código de la interfaz `QuestionRepository` es el siguiente:

```php
<?php // src/Forum/Question/Repository/QuestionRepository.php

namespace Forum\Question\Repository;

use Forum\Question\Question;

interface QuestionRepository
{
    /**
     * Agrega o Actualiza una pregunta en el repositorio.
     *
     * @param Question $question
     */
    public function save(Question $question);

    /**
     * Busca una pregunta por su id y la retorna.
     * Si no existe, rentorna null
     * @param $id
     * @return Question|null
     */
    public function find($id);

    /**
     * Retorna un arreglo de preguntas ordenados de forma descendiente.
     *
     * @return Question[]|array
     */
    public function findAll();
}
```

### La clase DbQuestionRepository

Para efectos de estos tutoriales las implementaciones de los repositorios utilizarán [Doctrine DBAL](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/) para comunicarse con la base de datos, y asi aplicar buenas prácticas de creación de código limpio.

```php
<?php // src/Forum/Question/Repository/DbQuestionRepository.php

namespace Forum\Question\Repository;

use Doctrine\DBAL\Connection;
use Forum\Question\Factory\QuestionFactory;
use Forum\Question\Question;

class DbQuestionRepository implements QuestionRepository
{
    /**
     * @var Connection
     */
    private $connection;

    /**
     * @var QuestionFactory
     */
    private $factory;

    public function __construct(Connection $connection, QuestionFactory $factory)
    {
        $this->connection = $connection;
        $this->factory = $factory;
    }

    public function save(Question $question)
    {
        if ($question->getId()) {
            $this->update($question);
        } else {
            $this->create($question);
        }
    }

    public function find($id)
    {
        $data = $this->connection->fetchAssoc('SELECT * FROM questions WHERE id = ?', [$id]);

        return $this->factory->createFromArray($data);
    }

    public function findAll()
    {
        $data = $this->connection->fetchAll('SELECT * FROM questions ORDER BY id DESC');

        $questions = [];

        foreach ($data as $item) {
            $questions[] = $this->factory->createFromArray($item);
        }

        return $questions;
    }

    private function create(Question $question)
    {
        $id = $this->connection->insert(
            'questions',
            $this->toArray($question),
            ['created' => 'datetime'] // Para que Doctrine convierte el \Datetime a string
        );

        $question->setId($id);
    }

    private function update(Question $question)
    {
        $this->connection->update(
            'questions',
            $this->toArray($question),
            ['id' => $question->getId()],
            ['created' => 'datetime'] // Para que Doctrine convierte el \Datetime a string
        );
    }

    private function toArray(Question $question)
    {
        return [
            'title' => $question->getTitle(),
            'description' => $question->getDescription(),
            'author' => $question->getAuthor(),
            'created' => $question->getCreatedAt(),
            'resolved' => $question->isResolved(),
        ];
    }
}
```

Como se puede ver, el repositorio tiene dos dependencias, la clase `Doctrine\DBAL\Connection` de Doctrine y la clase `Forum\Question\Factory\QuestionFactory`. La primera nos permite comunicarnos con la Base de datos, y la segunda nos permite crear objetos de tipo `Forum\Question\Question` en base a un array.

### La clase QuestionFactory

Esta clase tiene como finalidad convertir un arreglo (Que es lo que obtenemos al consultar la base de datos con Doctrine DBAL) en una instancia de la clase `Forum\Question\Question`. Su contenido es:

```php
<?php // src/Forum/Question/Factory/QuestionFactory.php

namespace Forum\Question\Factory;

use Forum\Question\Question;

class QuestionFactory
{
    /**
     * Crea un objeto Question a partir de un arreglo
     *
     * @param array $data
     * @return Question
     */
    public function createFromArray($data)
    {
        $question = new Question(
            $data['title'],
            $data['description'],
            $data['author'],
            new \DateTime($data['created'])
        );

        $question->setId((int)$data['id']);
        $data['resolved'] and $question->markAsResolved();

        return $question;
    }
}
```

<hr>

## Registrando las clases como servicios

Gracias a que estamos usando Symfony disponemos de un contenedor de clases, por lo que vamos a registrar el repositorio y sus dependencias como un servicio.

##### Creando el app/config/parameters.yml

Siguiendo el estandar de los proyectos en Symfony vamos a crear un archivo YAML para nuestros parametros:

```yaml
parameters:
    database_host: "127.0.0.1"
    database_driver: "pdo_mysql"
    database_name: "micro-foro"
    database_user: "root"
    database_password: ~
```

##### Creando el app/config/services.yml

En este archivo se van a ir registrando los servicios mientras se vaya avanzando en la construcción de la aplicación:

```yaml
services:
    # Clase necesaria para la conexión DBAL
    database.configuration:
        public: false
        class: Doctrine\DBAL\Configuration

    # Representa la conexión con la base de datos
    database.connection:
        public: false
        class: Doctrine\DBAL\Connection
        factory: [Doctrine\DBAL\DriverManager, getConnection]
        arguments:
            -   host: "%database_host%"
                dbname: "%database_name%"
                user: "%database_user%"
                password: "%database_password%"
                driver: "%database_driver%"
            - "@database.configuration"

    # Representa la implementación de Forum\Question\Repository\QuestionRepository
    repository.question:
        class: Forum\Question\Repository\DbQuestionRepository
        autowire: true # el autowire activa la inyección de dependencias automática.
        # los argumentos son inyectados de forma automática gracias al autowiring
```

Inicialmente tenemos estos tres servicios: `database.configuration`, `database.connection` y `repository.question`, los cuales nos permiten comunicarnos con la base de datos y administrar las preguntas.

Cabe destacar que el repositorio utiliza una funcionalidad llamada [autowire](http://symfony.com/doc/current/components/dependency_injection/autowiring.html), que se encarga de pasarle las dependencias al repositorio de forma automática en base a los tipos de datos esperados por nuestra clase en su constructor.

##### Cambios en el config.yml

Por último en la parte de configuración vamos a registrar los nuevos archivos yaml en el config.yml, además vamos a activar algunos componenets que serán necesarios para validar y crear formularios de las preguntas:

```yaml
imports:
    - { resource: parameters.yml }
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

Se agrega un archivo de temas de formulario en la configuración de twig. El contenido de dicho archivo es el siguiente:

__app/Resources/views/form/fields.html.twig__

{% raw %}
```twig
{# app/Resources/views/form/fields.html.twig #}
{% use 'form_div_layout.html.twig' %}

{% block form_row %}
    <div class="form-row{% if (not compound or force_error|default(false)) and not valid %} has-error{% endif %}">
        {{- form_label(form) -}}
        {{- form_widget(form) -}}
        {{- form_errors(form) -}}
    </div>
{% endblock %}

{%- block form_errors -%}
    {%- if errors|length > 0 -%}
        <ul class="form-errors">
            {%- for error in errors -%}
                <li>{{ error.message }}</li>
            {%- endfor -%}
        </ul>
    {%- endif -%}
{%- endblock form_errors -%}

{% block form_widget_simple -%}
    {% if type is not defined or type not in ['file', 'hidden'] %}
        {%- set attr = attr|merge({class: (attr.class|default('') ~ ' form-widget')|trim}) -%}
    {% endif %}
    {{- parent() -}}
{%- endblock form_widget_simple %}

{% block textarea_widget -%}
    {% set attr = attr|merge({class: (attr.class|default('') ~ ' form-widget')|trim}) %}
    {{- parent() -}}
{%- endblock textarea_widget %}
```
{% endraw %}

- - -

## Creando y Listando Preguntas

Con las clases del Negocio ya creadas, ahora vamos a crear las clases y archivos propias del módulo de creación y listado de preguntas. Los archivos que vamos a crear son:

    app/Resources/views/question/new.html.twig
    app/Resources/views/question/list.html.twig
    src/App/Controller/QuestionController.php
    src/App/Form/QuestionType.php

Comenzaremos por la clase de formulario:

##### El Formulario __src/App/Form/QuestionType.php__

```php
<?php  // src/App/Form/QuestionType.php

namespace App\Form;

use Forum\Question\Question;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextareaType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\FormInterface;
use Symfony\Component\OptionsResolver\Options;
use Symfony\Component\OptionsResolver\OptionsResolver;

class QuestionType extends AbstractType
{

    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder->add('title', TextType::class);
        $builder->add('description', TextareaType::class);
    }

    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefaults([
            'data_class' => Question::class,
            'empty_data' => function (Options $options) {
                return function (FormInterface $form) use ($options) {
                    return new Question(
                        $form['title']->getData(),
                        $form['description']->getData(),
                        $options['author']
                    );
                };
            },
        ]);

        $resolver->setRequired('author');
        $resolver->setAllowedTypes('author', 'string');
    }
}
```

Nuestro formulario posee dos campos **title** y **description*, además requiere de una opción `author` que será el nombre del usuario que hace la pregunta.

Hemos definido la opción `empty_data` debido a que la clase `Forum\Question\Question` requiere que le pasemos algunos datos en su constructor, y la forma de hacerlo es con [dicha opción](http://symfony.com/doc/current/form/use_empty_data.html#option-1-instantiate-a-new-class).

##### El __src/App/Controller/QuestionController.php__

```php
<?php // src/App/Controller/QuestionController.php

namespace App\Controller;

use App\Form\QuestionType;
use Forum\Question\Question;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\HttpFoundation\Request;

class QuestionController extends Controller
{
    /**
     * @Route("/", name="question_list")
     */
    public function listAction()
    {
        $questions = $this->get('repository.question')->findAll();

        return $this->render('question/list.html.twig', [
            'questions' => $questions,
        ]);
    }

    /**
     * @Route("/new", name="question_create")
     */
    public function newAction(Request $request)
    {
        $form = $this->createForm(QuestionType::class, null, [
            'author' => 'test@test.com',
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
}
```

El controlador posee dos acciones `"/"` y `"/new"`, la primera se encarga de listar las preguntas y la segunda nos permite crearlas.

##### Las vistas

Cada acción va a cargar una vista, y el contenido de cada una es el siguiente: 

 __app/Resources/views/question/list.html.twig__

{% raw %}
```twig
{% extends 'base.html.twig' %}

{% block page_header %}
    Preguntas Recientes
    <span class="pull-right">
        <a href="{{ path('question_create') }}" class="button primary">Hacer una Pregunta</a>
    </span>
{% endblock %}

{% block content %}
    {% for question in questions %}
        <div class="question">
            <h3><a href="{{ path('question_list') }}">{{ question.title }}</a></h3>
            <div class="question-content">{{ question.description }}</div>
            <p class="question-footer">
                {{ question.author }} - {{ question.createdAt|date }}
            </p>
        </div>
    {% else %}
        <h2>No hay preguntas creadas</h2>
    {% endfor %}
{% endblock %}

```
{% endraw %}

 __app/Resources/views/question/new.html.twig__

{% raw %}
```twig
{% extends 'base.html.twig' %}

{% block page_header %}
    Crear Pregunta
{% endblock %}

{% block content %}
    {{ form_start(form, {attr: {novalidate: 'novalidate'}}) }}
    {{ form_row(form.title, { label: 'Título de la Pregunta' }) }}
    {{ form_row(form.description, { label: 'Descripción de la Pregunta', attr: { class:'description-widget' } }) }}
    <div class="form-actions">
        <input type="submit">
        <a href="{{ path('question_list') }}" class="button">Volver</a>
    </div>
    {{ form_end(form) }}
{% endblock %}
```
{% endraw %}

Eso es todo, sí ahora iniciamos el servidor con `php -S localhost:8000 -t web`, veremos una pantalla como la siguiente:

##### Pantallas de la aplicación

![Listado de Preguntas]({{ site.url }}/assets/images/micro-foro/2-listado-vacio.png)

Esta pantalla muestra el listado de Preguntas, que ahora mismo está vacio. Si presionamos el botón <img src="{{ site.url }}/assets/images/micro-foro/2-boton-hacer-pregunta.png" class="img-inline" alt="Hacer una Pregunta" style="max-width: 130px;" /> Vamos a ver un formulario como el siguiente: 

![Formulario de pregunta]({{ site.url }}/assets/images/micro-foro/2-form-valido.png)

Nuestro formulario posee dos campos, uno para el título y otro para la descripción de la pregunta. Ingresando una data de prueba y Presionando el botón de <img src="{{ site.url }}/assets/images/micro-foro/2-boton-form.png" class="img-inline" alt="Hacer una Pregunta" style="max-width: 100px;" /> La aplicación creará la pregunta y nos va a redirigir al listado de preguntas, donde podremos visualizar el registro recien creado:

<!--
![Listado de Preguntas]({{ site.url }}/assets/images/micro-foro/2-form.png)
![Listado de Preguntas]({{ site.url }}/assets/images/micro-foro/2-form-invalido.png)
-->

![Listado de Preguntas]({{ site.url }}/assets/images/micro-foro/2-listado.png)

Como se puede apreciar, ahora en el listado aparece la pregunta recien creada.

Bueno, hasta acá la segunda parte de la construcción del micro foro. [Más adelante](/2016/08/30/foro-micro-kernel-trait-symfony-parte-3.html) seguiremos con la administración de las preguntas, añadiendo seguridad al sitio web para relacionar las preguntas que se crean con el usuario logueado.

[Descargar o visualizar el Proyecto](https://github.com/manuelj555/foro-microkerneltrait/releases/tag/foro-parte-2)