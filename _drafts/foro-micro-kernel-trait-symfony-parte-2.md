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

Lo primero que haremos será crear una base de datos para conectarnos a ella. Para efectos del ejemplo se decidió trabajar con una base de datos Mysql. Llamaremos a la base de datos **micro-blog** y crearemos la siguiente tabla:

{% highlight sql linenos %}
CREATE TABLE `questions` (
    `id` INT (11) NOT NULL AUTO_INCREMENT,
    `title` VARCHAR (255) NOT NULL,
    `description` text NOT NULL,
    `author` VARCHAR (255) NOT NULL,
    `created` datetime NOT NULL,
    `resolved` bit (1) NOT NULL,
     PRIMARY KEY (`id`)
) ENGINE = INNODB
{% endhighlight %}

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

{% highlight php linenos %}
<?php

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
{% endhighlight %}

La clase posee un atributo por cada propiedad de la pregunta, además se agregan algunas [Registricciones de Validación](http://symfony.com/doc/current/validation.html#constraints), aprovenchando el [Validador de Symfony](http://symfony.com/doc/current/validation.html).

### La clase QuestionRepository

Debido a que necesitamos de alguna manera obtener y persistir las preguntas en la aplicación, vamos hacer uso del [patrón repositorio](http://stackoverflow.com/questions/16176990/proper-repository-pattern-design-in-php) el cual nos permite mapear los modelos contra una fuente de datos que puede ser una base de datos, una api rest, la sesión, etc.

Teniendo en cuenta que debemos poder crear, listar y actualizar preguntas en la base de datos, vamos a definir los siguientes métodos:

 * **save**: Crea o actualiza una pregunta en la base de datos.
 * **find**: Obtiene una pregunta en base a su id.
 * **findAll**: Obtiene las preguntas ordenadas de forma descendiente.

El código de la interfaz `QuestionRepository` es el siguiente:

{% highlight php linenos %}
<?php

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
{% endhighlight %}

### La clase DbQuestionRepository

Para efectos de estos tutoriales las implementaciones de los repositorios utilizarán [Doctrine DBAL](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/) para comunicarse con la base de datos, y asi aplicar buenas prácticas de creación de código limpio.

{% highlight php linenos %}
<?php

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
{% endhighlight %}

Como se puede ver, el repositorio tiene dos dependencias, la clase `Doctrine\DBAL\Connection` de Doctrine y la clase `Forum\Question\Factory\QuestionFactory`. La primera nos permite comunicarnos con la Base de datos, y la segunda nos permite crear objetos de tipo `Forum\Question\Question` en base a un array.

### La clase QuestionFactory

Esta clase tiene como finalidad convertir un arreglo (Que es lo que obtenemos al consultar la base de datos con Doctrine DBAL) en una instancia de la clase `Forum\Question\Question`. Su contenido es:

{% highlight php linenos %}
<?php

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
{% endhighlight %}

<hr>

## Registrando las clases como servicios

Gracias a que estamos usando Symfony disponemos de un contenedor de clases, por lo que vamos a registrar el repositorio y sus dependencias como un servicio.

##### Creando el app/config/parameters.yml

Siguiendo el estandar de los proyectos en Symfony vamos a crear un archivo YAML para nuestros parametros:

{% highlight yaml linenos %}
parameters:
    database_host: 127.0.0.1
    database_driver: pdo_mysql
    database_name: micro-foro
    database_user: root
    database_password: ~
{% endhighlight %}

##### Creando el app/config/services.yml

En este archivo se van a ir registrando los servicios mientras se vaya avanzando en la construcción de la aplicación:

{% highlight yaml linenos %}
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

{% endhighlight %}

Inicialmente tenemos estos tres servicios: `database.configuration`, `database.connection` y `repository.question`, los cuales nos permiten comunicarnos con la base de datos y administrar las preguntas.

Cabe destacar que el repositorio utiliza una funcionalidad llamada [autowire](http://symfony.com/doc/current/components/dependency_injection/autowiring.html), que se encarga de pasarle las dependencias al repositorio de forma automática en base a los tipos de datos esperados por nuestra clase.

##### Cambios en el config.yml

Por último en la parte de configuración vamos a registrar los nuevos archivos yaml en el config.yml, además vamos a activar algunos componenets que serán necesarios para validar y crear formularios de las preguntas:

{% highlight yaml linenos %}
imports:
    - { resource: parameters.yml }
    - { resource: services.yml }

framework:
    secret: StringAleatorio
    templating:
        engines: ['twig']
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
{% endhighlight %}

<hr>

## Creando una Pregunta