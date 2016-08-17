---
layout: post
title: Creando Validadores en Symfony
---

Las [validaciones en Symfony](http://symfony.com/doc/current/book/validation.html) se trabajan con el componente [validator](http://symfony.com/components/Validator), y realmente son muy sencillas de definir ya sea en yaml, xml o usando anotaciones.

Pero aveces cuando necesitamos realizar una validación algo compleja y que no viene entre los diferentes [Constraints predefinidos en el componente](http://symfony.com/doc/current/reference/constraints.html), optamos por hacer esa validación desde otro lado, ya sea un listener de formulario, en el controlador, en una clase que administra la entidad, etc, sin indagar antes en la posibilidad que brinda el validador de agregar nuestros [propios constraints](http://symfony.com/doc/current/cookbook/validation/custom_constraint.html).

En este post, veremos un ejemplo donde crearemos restricciones personalizadas.


## Validando Códigos de Inscripción

Supongamos que tenemos una aplicación que permite el registro de usuarios online, y estos, a la hora de registrarse deben tener un código de registro que la empresa o institución previamente les ha aportado.

Entonces la idea es que cuando un usuario se intente registrar, verifiquemos que el código que ingrese sea un código existente en la base de datos de la aplicación.

Como nos podemos dar cuenta, el componente de validación no tiene una Constraint para un caso como este, por lo que nuestra opción será crear una.

### Entidad User

La entidad usuario tendrá tres atributos, **name**, **email** y **code**, ninguno puede ser nulo, el email debe ser un correo válido y el código debe ser uno existente en la base de datos.

{% highlight php linenos %}
<?php

namespace MyBundle\Entity;

use Symfony\Component\Validator\Constraints as Assert;

class User
{
	/**
	 * @Assert\NotBlank(message="el campo no puede estar vacío")
	 */
    protected $name;

    /**
	 * @Assert\NotBlank(message="el campo no puede estar vacío")
	 * @Assert\Email(message="el campo debe ser un correo válido")
	 */
    protected $email;

    /**
	 * @Assert\NotBlank(message="el campo no puede estar vacío")
	 */
    protected $code;

    // ...getters y setters
}

{% endhighlight %}

### Entidad Code

La entidad Code será la que nos permita obtener y buscar los códigos en la Base de Datos.

{% highlight php linenos %}
<?php

namespace MyBundle\Entity;

class Code
{
    protected $code;
}
{% endhighlight %}

### Creando el Constraint

Leyendo la documentación sobre la creación de [constraints propios](http://symfony.com/doc/current/cookbook/validation/custom_constraint.html) vemos que debemos crear basicamente dos clases, la primera es un constraint, que es el usado en la definicion de las reestricciones en los modelos, y la otra clase es la que realiza la validación.

Nuestro constraint se llamará **ValidCode**, y el validador que devolverá será un string (**valid_code**) que representa al validador definido como servicio en la aplicación:

{% highlight php linenos %}
<?php

namespace MyBundle\Validator\Constraint;

use Symfony\Component\Validator\Constraint;

/**
 * @Annotation
 * @Target({"PROPERTY"})
 */
class ValidCode extends Constraint
{
    public $message = 'El código es incorrecto';

    public function validatedBy()
    {
        return 'valid_code';
        //este string representa un servicio validador definido en el proyecto
    }

}
{% endhighlight %}

Ahora debemos crear la clase que realizará el proceso de validación:

### Creando el Validador

La clase **ValidCodeValidator** será la encargada de hacer el proceso de validación, y si al validar no se encuentra el códigó en la base de datos, se añadirá una violación al modelo:

{% highlight php linenos %}
<?php

namespace MyBundle\Validator;

use Symfony\Component\Validator\Constraint;

class ValidCodeValidator extends ConstraintValidator
{
    protected $codeRepository;

    function __construct($codeRepository)
    {
        $this->codeRepository = $codeRepository;
    }

    /**
     * Verifica que el código pasado seá uno existente en la base de datos.
     */
    public function validate($value, Constraint $constraint)
    {
        if (!($code = $this->codeRepository->findOneByCode($value))){
            $this->context->buildViolation($constraint->message)->addViolation();
        }
    }
}
{% endhighlight %}

### Registrando el Validador

{% highlight yaml linenos %}
#  MyBundle/Resources/config/services.yml
services:
    my_bundle.code_repository:
        class: Doctrine\ORM\EntityRepository
        factory_service: doctrine.orm.entity_manager
        factory_method: getRepository
        arguments: [MyBundle:Code]

    my_bundle.validator.valid_code
        class: MyBundle\Validator\ValidCode
        arguments: [@my_bundle.code_repository]
        tags:
            - { name: validator.constraint_validator, alias: valid_code }
{% endhighlight %}

Definimos el servicio **my\_bundle.validator.valid\_code** y le creamos el [tag](http://symfony.com/doc/current/cookbook/validation/custom_constraint.html#constraint-validators-with-dependencies) **validator.constraint\_validator**, para que el componente validador lo reconosca y lo pueda cargar mediante el alias **valid_code** que es el retornado en el método validateBy de la clase ValidCode.

### Usando la Constraint

{% highlight php linenos %}
<?php

namespace MyBundle\Entity;

use Symfony\Component\Validator\Constraints as Assert;
use MyBundle\Validator\Constraint\ValidCode;

class User
{
	//......

    /**
	 * @Assert\NotBlank(message="el campo no puede estar vacío")
	 *
	 * @ValidCode()
	 */
    protected $code;

    // ...getters y setters
}
{% endhighlight %}

Añadiendo la anotación **@ValidCode()** (y el use correspondiente) al validar la entidad, el validador verificará que el valor del atributo $code sea un código existente en la base de datos, en el caso en donde el código no exista, el validador nos devolverá un error para ese atributo.

#### Conclusión

Puede ser muy util crearnos nuestras validaciones, ya que permiten tener bien organizados los procesos de validación de datos, y además ayudan a la reutilización de los mismos en otros objetos y entidades.

Tambien es importante ver que no todos los validadores deben ser declarados como servicios, ya que en algunos casos la validacion no requiere de dependencias externas como doctrine u otros, por lo que el método validateBy puede devolver la instancia de la clase validadora directamente. Lo mejor es indagar sobre este importante componente en la [documentación](http://symfony.com/doc/current/cookbook/validation/custom_constraint.html).