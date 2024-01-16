# Routing

## Configuration (YAML and PHP attributes)

Base config (automatically provided by flex) 
```yaml
# config/routes/attributes.yaml
controllers:
    resource:
        path: ../../src/Controller/
        namespace: App\Controller
    type: attribute

kernel:
    resource: App\Kernel
    type: attribute
```

**Query string of a URL is not considered when matching routes** (query parameters like /blog?foo=bar et blog/?foo=bar&bar=foo match also the base url /blog)

## Parameters

You can get the route name and route parameters with the Request object :
```php
$routeName = $request->attributes->get('_route');
$routeParameters = $request->attributes->get('_route_params');
// use this to get all the available attributes (not only routing ones):
$allAttributes = $request->attributes->all();
```
In services, you'll have to use [RequestStack](https://symfony.com/doc/current/service_container/request.html) object. In templates we use the global app variable to get current route name (`app.current_route`) and its parameters (`app.current_route_parameters`)

### Validation

`requirements` key can be used to restrict a url parameter () :
```php
#[Route('/blog/{page}', name: 'blog_list', requirements: ['page' => '\d+'])]
    public function list(int $page): Response
    {
        // ...
    }
```
* route requirements and route paths can include [configuration parameters](https://symfony.com/doc/current/configuration.html#configuration-parameters)
* requirements can be inlined with that syntax `{parameter_name<requirements>}`

### Optional Parameters

**A parameter added to a route must have a value**, so if route's url is '/blog/{page}' /blog will not match this route, unless $page has a default value in the controller method.
* use {page?} to allow null values (with `?int $page` in the method parameters)
* use /blog/{!page} to not match /blog despites $page default value

### Parameter Conversion

If the parameter is type hinted as an object in the controller, and if it matches the name of a property of this object in the url, a database request will be made for that object.
the #[MapEntity] attribute can be used to customize the query.

### Special Parameters

* `_controller`
* `_format` : set the request format of the Request object (e.g. a json format translates into a Content-Type of application/json).
* `_fragment` : (optional last part of a url beginning with #)
* `_locale`

### Slash Characters in Route Parameters

If the route includes the special {_format} parameter, you shouldn't use the .+ requirement for the parameters that allow slashes. For example, if the pattern is /share/{token}.{_format} and {token} allows any character, the /share/foo/bar.json URL will consider foo/bar.json as the token and the format will be empty. This can be solved by replacing the .+ requirement by [^.]+ to allow any character except dots.

### Prefix Routes

If a prefix route defines an empty path, symfony will add a trailing slash (an empty path prefixed with /blog will give /blog/). To avoid it use `trailing_slash_on_root` option.

## Generate URL parameters

### In Controllers

You can generate URLs with the generateUrl() method (from AbstractController). Parameters which are not parts of the URL definition are included in the generated url as a query string.

### In Services

by autowiring `Symfony\Component\Routing\Generator\UrlGeneratorInterface` in the service constructor to use the generate() method

### In templates

with the twig `path()` methos

### In Commands

Commands are not executed in HTTP context, so the `default_uri` option must be used
```yaml
# config/packages/routing.yaml
framework:
    router:
        # ...
        default_uri: 'https://example.org/my/path/'
```

## Domain name matching

Routes can configure a host option to require that the HTTP host of the incoming requests matches some specific value.
This option can include parameters that can be validated
```php
// src/Controller/MainController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;

class MainController extends AbstractController
{
    #[Route(
        '/',
        name: 'mobile_homepage',
        host: '{subdomain}.example.com',
        defaults: ['subdomain' => 'm'],
        requirements: ['subdomain' => 'm|mobile'],
    )]
    public function mobileHomepage(): Response
    {
        // ...
    }

    #[Route('/', name: 'homepage')]
    public function homepage(): Response
    {
        // ...
    }
}
```

## Conditional Request Matching

You can use any regular expression with the `condition` option to conditionnally match an url. 
These three vars can be used :
* `context` : An instance of [RequestContext](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/Routing/RequestContext.php), which holds the most fundamental information about the route being matched.
* `request` : The Symfony Request object that represents the current request.
* `params` : An array of matched route parameters for the current route.

These functions :
* `env(string $name)` : Returns the value of a variable using Environment Variable Processors
* `service(string $alias)` : Returns a routing condition service

The service has to add the #[AsRoutingConditionService] attribute or routing.condition_service tag

## HTTP methods matching

With the `methods` options in Route attribute
HTML form only support GET and POST methods, but symfony form adds automatically a hidden field (e.g. <input type="hidden" name="_method" value="PUT">) if `framework.http_method_override` on true.

## Debugging

Two commands

* `bin/console debug:router` : lists all our application routes and can take a route's name as an argument
* `bin/console router:match` : return the route which matches a given url

