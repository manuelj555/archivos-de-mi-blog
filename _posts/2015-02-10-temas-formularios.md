---
layout: post
title: Trabajando con los temas de Formularios en Symfony
---

En las aplicaciones construidas con symfony siempre es necesario que los formularios tengan un aspecto visual adaptado al diseño general de la página.

Muchas veces por desconocimiento optamos por renderizar cada elemento de los formularios de forma separada y manual, esto con el fin de lograr que visualmente el formulario se vea según los requerimientos.

### La forma Manual

Un ejemplo muy común de lo expuesto es un formulario como el siguiente:

{% highlight jinja linenos %}
{% raw %}
{{ form_start(form) }}
    <div class="form_errors">
        {{ form_errors(form) }}
    </div>
    <div class="form_content">
        <div class="form-group{% if not form.firstName.vars.valid %} with-errors{% endif %}">
            {{ form_label(form.firstName, { label_attr: { class: 'f-lbl' }}) }}
            <div class="f-item">
                {{ form_errors(form.firstName) }}
                {{ form_widget(form.firstName, { attr: { class: 'f-element' }}) }}
            </div>
        </div>
        <div class="form-group{% if not form.lastName.vars.valid %} with-errors{% endif %}">
            {{ form_label(form.lastName, { label_attr: { class: 'f-lbl' }}) }}
            <div class="f-item">
                {{ form_errors(form.lastName) }}
                {{ form_widget(form.lastName, { attr: { class: 'f-element' }}) }}
            </div>
        </div>
        <div class="form-group{% if not form.email.vars.valid %} with-errors{% endif %}">
            {{ form_label(form.email, { label_attr: { class: 'f-lbl' }}) }}
            <div class="f-item">
                {{ form_errors(form.email) }}
                {{ form_widget(form.email, { attr: { class: 'f-element' }}) }}
            </div>
        </div>
        <div class="form-group{% if not form.phone.vars.valid %} with-errors{% endif %}">
            {{ form_label(form.phone, { label_attr: { class: 'f-lbl' }}) }}
            <div class="f-item">
                {{ form_errors(form.phone) }}
                {{ form_widget(form.phone, { attr: { class: 'f-element' }}) }}
            </div>
        </div>
        <div class="form-group{% if not form.country.vars.valid %} with-errors{% endif %}">
            {{ form_label(form.country, { label_attr: { class: 'f-lbl' }}) }}
            <div class="f-item">
                {{ form_errors(form.country) }}
                {{ form_widget(form.country, { attr: { class: 'f-select' }}) }}
            </div>
        </div>
    </div>
{{ form_end(form) }}
{% endraw %}
{% endhighlight %}

Como se puede apreciar, cada elemento del formulario es renderizado dentro de una estructura html, donde se imprimen, el label, los errores y el elemento de manera separada y pasandole las clases necesarias para que tomen el diseño correspondiente, y así lograr el aspecto visual requerido.

Imaginemos ahora, que nuestra aplicación tiene unos 10 o más formularios, quiere decir que en cada página estamos usando la misma técnica para lograr que todos los forms se vean iguales.

Hacer esto no es nada óptimo, debido a que si más adelante necesitamos cambiar el aspecto de los forms, ya sea modificando la estructura html o clases usadas por los elementos, deberemos actualizar todas las vistas de los formularios para obtener el ajuste.

Es por ello que symfony implementa un sistema de temas de formularios, que permite configurar el aspecto de cada elemento de los forms de manera global y/o especifica.

### Creando los Form Themes

Veamos los pasos a seguir para lograr que el código anterior implemente los temas de formularios.

###### El archivo de temas:

Este archivo contiene una serie de bloques twig, donde el interior de cada uno es código html que será renderizado para cada elementos de formulario, inputs, labels, textareas, selects, checboxes, errores, entre otros.

{% highlight jinja linenos %}
{% raw %}
{# app/Resources/views/Form/fields.html.twig #}
{# Extendemos del layout por defecto para reutilizar widgets usando parent() #}
{% extends 'form_div_layout.html.twig' %}


{# Bloque que renderiza cualquier elemento de tipo <input ... #}

{% block form_widget_simple -%}
    {% if type is not defined or 'file' != type %}
        {# Seteamos la clase f-elemento por defecto al <input, y al mismo tiempo permitimos que se añadan clases adicionales #}
        {% set attr = attr|merge({class: (attr.class|default('') ~ ' f-element')|trim}) %}
    {% endif %}
    {{- parent() -}}
{%- endblock form_widget_simple %}


{# Bloque que renderiza los textareas #}

{% block textarea_widget -%}
    {% set attr = attr|merge({class: (attr.class|default('') ~ ' f-element')|trim}) %}
    {{- parent() -}}
{%- endblock textarea_widget %}


{# Bloque que renderiza los selects #}

{% block choice_widget_collapsed -%}
    {% set attr = attr|merge({class: (attr.class|default('') ~ ' f-select')|trim}) %}
    {{- parent() -}}
{%- endblock %}


{# Bloque que renderiza los labels #}

{% block form_label -%}
    {% set label_attr = label_attr|merge({class: (label_attr.class|default('') ~ ' f-lbl')|trim}) %}
    {{- parent() -}}
{%- endblock form_label %}


{# Bloque que renderiza un campo de formulario completo, label, widget y errores #}

{% block form_row -%}
    <div class="form-group{% if not valid %} with-errors{% endif %}">
        {{ form_label(form) }}
        <div class="f-item">
            {{ form_errors(form) }}
            {{ form_widget(form) }}
        </div>
    </div>
{%- endblock %}

{% endraw %}
{% endhighlight %}

Así hemos creado una serie de bloques que nos van a permitir reutilizar diseños en todos nuestros formularios.

###### Nombres de los Bloques

Existe una convención para el nombre de los bloques, y dependerá un poco de intuición y experiencia lograr dominar este punto, en la documentación oficial tratan esta parte con [más detalle](http://symfony.com/doc/current/cookbook/form/form_customization.html)

Como regla general, el bloque tiene por nombre el tipo de campo, **text, file, textarea, checkbox, email, ...** segido de los posibles sufijos **_row** o **_widget**, la diferencia entre la terminación _row y _widget es que el primero renderizará todo el campo (label, widget y errores) mientras el segundo solo renderiza el propio widget.

Cuando renderizamos un campo de formulario llamando a la funcion **form_row(form.element)** el bloque llamado será el que termine en **_row** para el campo especifico, mientras que cuando usamos **form_widget(form.element)** el bloque ejecutado será el que contenga el sufijo **_widget**.

La función **form_label(form.element)** invoca al unico bloque existente para los labels llamado **form_label**.

Algo importante a destacar es que cuando no existe un bloque definido para un campo especifico, se intenta llamar al bloque para el tipo de formulario padre, es por eso que el bloque **form_row** sirve para todos los campos de formulario que no definen su propio bloque row.

### Usando los Form Themes

Ahora que tenemos creado nuestro archivo de temas, debemos incluirlo en los formularios, para que tomen el diseño adecuado.

Para lograrlo existen dos formas, la primera es hacerlo de manera global, agregando el tema en la configuración de twig:

###### Temas Globales

{% highlight yaml linenos %}
# app/config/config.yml
twig:
    debug:            "%kernel.debug%"
    strict_variables: "%kernel.debug%"
    form:
        resources:
            - "Form/fields.html.twig" # Acá estamos añadiendo nuestro tema de forma global.
{% endhighlight %}

Haciendo esto, todos los formularios de la aplicación ahora serán renderizados utilizando los bloques de tema definidos en el twig.

###### Temas para formularios especificos

Cuando solo queremos aplicar cierto tema a un formulario especifico, lo hacemos de la siguiente manera:

{% highlight jinja linenos %}
{% raw %}
{# app/Resources/views/my_form.html.twig #}

{% form_theme form 'Form/fields.html.twig' %}

...

{% endraw %}
{% endhighlight %}

Existe una prioridad a la hora de determinar cual tema utilizar para el renderizado, la prioridad comienza con los temas definidos en el mismo archivo twig con el tag form_theme, si el tema de cierto campo no existe allí, se busca en los archivos definidos en el config.yml, y por último se busca en form\_div\_layout.html.twig del componente de formularios.

### Simplificando la impresión de los formularios

Ahora que hemos establecido los temas para los formularios, podemos simplificar el código de las vistas que imprimen los forms:

{% highlight jinja linenos %}
{% raw %}

{# {% form_theme form 'Form/fields.html.twig' %} descomentar si el tema no es usado de forma global #}

{{ form_start(form) }}
    <div class="form_errors">
        {{ form_errors(form) }}
    </div>
    <div class="form_content">
        {{ form_row(form.firstName) }}
        {{ form_row(form.lastName) }}
        {{ form_row(form.email) }}
        {{ form_row(form.phone) }}
        {{ form_row(form.country) }}
    </div>
{{ form_end(form) }}
{% endraw %}
{% endhighlight %}

Como ven, se ha simplificado de manera extraordinaria el código necesario para renderizar el formulario, todo gracias al uso de los temas de formularios!.


Para más información consultar la documentación Oficial:

* [Doc Oficial](http://symfony.com/doc/current/cookbook/form/form_customization.html)
* [Entendiendo los Nombres de los Bloques](http://symfony.com/doc/current/cookbook/form/form_customization.html#cookbook-form-customization-sidebar)
* [Multiples temas para un mismo form](http://symfony.com/doc/current/cookbook/form/form_customization.html#multiple-templates)
* [Temas para campos Hijos](http://symfony.com/doc/current/cookbook/form/form_customization.html#child-forms)
* [Temas Globales](http://symfony.com/doc/current/cookbook/form/form_customization.html#making-application-wide-customizations)
