# Templating with Twig

## Auto escaping

auto escaping is enabled by default and can be disabled using the `raw` filter

## Template inheritance

* child template can't have code outside block
* we can use `extends` to inherit from another template
* we can use `include()` to include a twig fragment
* `parent()` method can be used to render the content from the parent block :

```twig
{% block sidebar %}
    <h3>Table Of Contents</h3>
    ...
    {{ parent() }}
{% endblock %}
```

## Global variables

### app variable

`app` variable is a context object from [AppVariable](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Bridge/Twig/AppVariable.php)
It gives access to :
* `app.user` : current user object | null
* `app.request` : current Request object
* `app.session` : current Session object | null
* `app.flashes` : array of flash messages stored in the session
* `app.environment`
* `app.debug` : true if in debug mode
* `app.token` : [TokenInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Security/Core/Authentication/Token/TokenInterface.php) object containing the security token
* `app.current_route` : The name of the route associated with the current request or null if no request is available (equivalent to `app.request.attributes.get('_route')`)
* `app.current_route_parameters` : An array with the parameters passed to the route of the current request or an empty array if no request is available (equivalent to `app.request.attributes.get('_route_params')`)
* `app.locale`
* `app.enabled_locales`

### managing global variables

* You can define your own global variables in twig.globals config options :
    ```yaml
    # config/packages/twig.yaml
    twig:
        # ...
        globals:
            # the value is the service's id
            uuid: '@App\Generator\UuidGenerator'
    ```
* You can also reference services by prefixing the service ID string with the @ character, which is the usual syntax to refer to services in container parameters. (They are not loaded lazily, as soons as twig is used, the service will be instantiated)

## Filters and Functions

[Native twig functions and filters](https://twig.symfony.com/doc/3.x/)

### Twig functions for Symfony

[Details on each function](https://symfony.com/doc/current/reference/twig_reference.html#functions)
* **render** : `{{ render(uri, options = []) }}`
* **render_esi** : `{{ render_esi(uri, options = []) }}`
* **fragment_uri** : `{{ fragment_uri(controller, absolute = false, strict = true, sign = true) }}`
* **controller** : `{{ controller(controller, attributes = [], query = []) }}`
* **asset** : `{{ asset(path, packageName = null) }}`
* **asset_version** : `{{ asset_version(packageName = null) }}`
* **csrf_token** : `{{ csrf_token(intention) }}`
* **is_granted** : `{{ is_granted(role, object = null, field = null) }}`
* **logout_path** : `{{ logout_path(key = null) }}`
* **logout_url** : `{{ logout_url(key = null) }}`
* **path** : `{{ path(route_name, route_parameters = [], relative = false) }}`
* **url** : `{{ url(route_name, route_parameters = [], relative = false) }}`
* **absolute_url** : `{{ absolute_url(path) }}`
* **relative_path** : `{{ relative_path(path) }}`
* **expression** : Creates an expression based on ExpressionLanguage component
* **impersonation_path** : `{{ impersonation_path(identifier) }}`
* **impersonation_url** : `{{ impersonation_url(identifier) }}`
* **impersonation_exit_path** : `{{ impersonation_exit_path(exitTo = null) }}`
* **impersonation_exit_url** : `{{ impersonation_exit_url(exitTo = null) }}`
* **t** : `{{ t(message, parameters = [], domain = 'messages')|trans }}`
* **importmap** : see [AssetMapper](https://symfony.com/doc/current/frontend/asset_mapper.html) component
* Forms related functions : 
    - form()
    - form_start()
    - form_end()
    - form_widget()
    - form_errors()
    - form_label()
    - form_help()
    - form_row()
    - form_rest()
    - field_name()
    - field_value()
    - field_label()
    - field_help()
    - field_errors()
    - field_choices()

The form_* functions render all html elements froma form field. To have full control, it's recommended to use the field_* methods that render only the value :
```twig
<input
    name="{{ field_name(form.username) }}"
    value="{{ field_value(form.username) }}"
    placeholder="{{ field_label(form.username) }}"
    class="form-control"
>

<select name="{{ field_name(form.country) }}" class="form-control">
    <option value="">{{ field_label(form.country) }}</option>

    {% for label, value in field_choices(form.country) %}
        <option value="{{ value }}">{{ label }}</option>
    {% endfor %}
</select>
```

### Twig filters for Symfony

[Details on each filter](https://symfony.com/doc/current/reference/twig_reference.html#filters)

* **humanize** : `{{ text|humanize }}`
* **trans** : `{{ message|trans(arguments = [], domain = null, locale = null) }}`
* **sanitize_html** : `{{ body|sanitize_html(sanitizer = "default") }}`
* **yaml_encode** : `{{ input|yaml_encode(inline = 0, dumpObjects = false) }}`
* **yaml_dump** : `{{ value|yaml_dump(inline = 0, dumpObjects = false) }}`
* **abbr_class** : `{{ class|abbr_class }}`
* **abbr_method** : `{{ method|abbr_method }}`
* **format_args** : `{{ args|format_args }}`
* **format_args_as_text** : `{{ args|format_args_as_text }}` (same as above but without generating `<em>` element)
* **file_excerpt** : `{{ file|file_excerpt(line, srcContext = 3) }}`
* **format_file** : `{{ file|format_file(line, text = null) }}`
* **format_file_from_text** : `{{ text|format_file_from_text }}`
* **file_link** : `{{ file|file_link(line) }}`
* **file_relative** : `{{ file|file_relative }}`
* **serialize** : `{{ object|serialize(format = 'json', context = []) }}`

## Tags (Symfony)

* **form_theme** : `{% form_theme form resources %}` - sets the resources to override form theme
* **trans** : `{% trans with vars from domain into locale %}{% endtrans %}`
* **trans_default_domain** : `{% trans_default_domain domain %}`
* **stopwatch** : `{% stopwatch 'event_name' %}...{% endstopwatch %}`
[See full details](https://symfony.com/doc/current/reference/twig_reference.html#tags)

## String concatenation and interpolation

1. Concatenation with the `~` operator :
```twig
{% set foo = 'hello' %}
{% set bar = 'world' %}
{{ foo ~ ' ' ~ bar ~ '!' }}
// output: hello world!
```

2. Concatenation with the `format` filter : 
```twig
{{ "%s %s!" | format(foo, bar) }}
// output: hello world!
```

3. String interpolation
```twig
{{ "#{foo} #{bar}!" }}
// output: hello world!

{{ "1 + 1 = #{1 + 1}" }}
// output: 1 + 1 = 2
```

## Debugging variables

There are two ways using `dump`, as afunction or as a tag :
```twig
{# templates/article/recent_list.html.twig #}
{# the contents of this variable are sent to the Web Debug Toolbar
   instead of dumping them inside the page contents #}
{% dump articles %}

{% for article in articles %}
    {# the contents of this variable are dumped inside the page contents
       and they are visible on the web page #}
    {{ dump(article) }}

    {# optionally, use named arguments to display them as labels next to
       the dumped contents #}
    {{ dump(blog_posts: articles, user: app.user) }}

    <a href="/article/{{ article.slug }}">
        {{ article.title }}
    </a>
{% endfor %}
```


