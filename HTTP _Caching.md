# HTTP Caching

Symfony cache system relies on HTTP cache as defined in **[RFC 7234 - Caching](https://tools.ietf.org/html/rfc7234)**, without reinventing new methodologies

## Cache Types (browser, proxies and reverse-proxies)

Browser cache is known as a **private cache** whereas proxy cache and reverse-proxy cache are **shared cache**.

[Details in MDN docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)

### Browser 

* Store data from navigation in a part of the user HDD.
* Store a personalized response for a specific user
* To avoid personalized responses to be cached in other cache than browser cache use `Cache-Control: private`

### Proxy Cache

* Usually not managed by the service developer, so must be controlled by appropriate HTTP  headers
* Using HTTPS, proxy caches in the path can only tunnel the response and can't behave as a cache. (Except using TLS bridge with a Certificate Authority, but this environment is heavily controlled, specially with some browsers that only allow signed certificate timestamp)

### Cache Gateway (Reverse Proxy)

* Gateway Cache is a type of cache sitting between client and our app (like Varnish or Squid)
* explicitly deployed by developers of the app, so unlike proxy caches, you can define a cahcing strategy specific to your needs and improve performance (i.e. load balancing)
* Symfony comes with a reverse proxy (a gateway cache) written in PHP (less efficient than a reverse proxy in C or Go) and a specific header `X-Symfony-Cache` used for debugging infos.
* To enable it in a prod env : 
```yaml
# config/packages/framework.yaml
when@prod:
    framework:
        http_cache: true
```
* You can switch to a faster reverse proxy (like Varnish) easily

## Expiration (Expires, Cache-Control)

* Used to cache an entire response for a specific amount of time
* Most efficient than Validation Cache (never hit the app until the response expires)

### Cache-Control

```php
use Symfony\Component\HttpKernel\Attribute\Cache;
// ...

#[Cache(public: true, maxage: 600)]
public function index(): Response
{
    // ...
}
```
equivalent to
```php
// sets the number of seconds after which the response
// should no longer be considered fresh by shared caches
$response->setPublic();
$response->setMaxAge(600);
```
Resulting header  : `Cache-Control: public, max-age=600`
* setSharedMaxAge() =/= setPublic() + setMaxAge() because s-maxage setting prohibits usage of `stale-if-error` scenario.

### Expires

* `Expires` header value is ignored if max-age or s-maxage are defined in `Cache-Control` header.
* setExpires() method convert the `DateTime` object to the GMT timezone
```php
use Symfony\Component\HttpKernel\Attribute\Cache;
// ...

#[Cache(expires: '+600 seconds')]
public function index(): Response
{
    // ...
}
```
equivalent to
```php
$date = new DateTime();
$date->modify('+600 seconds');

$response->setExpires($date);
```

## Validation (ETag, Last-Modified)

* You can use both expiration and validation. As expiration wins over validation, at every interval of expiration a validation process will be launched to know if a new response must be built
* Until response becomes invalid, server will return a 304 Not Modified without content telling the cache it can use its store version of the response 
* This model saves CPU only if determining if the response is still valid represents less work than generating the whole page again.

### Etag

* Optional HTTP header whose value is an arbitrary string set by our app
* Cache sets an `If-None-Match` header on the request (whose value will be the Etag of the original cached response) before sending the request to the app.
```php
// src/Controller/ArticleController.php
namespace App\Controller;

// ...
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

class ArticleController extends AbstractController
{
    public function show(string $articleSlug, Request $request): Response
    {
        // Get the minimum information to compute
        // the ETag or the Last-Modified value
        // (based on the Request, data is retrieved from
        // a database or a key-value store for instance)
        $article = ...;

        // create a Response with an ETag and/or a Last-Modified header
        $response = new Response();
        $response->setEtag($article->computeETag());
        $response->setLastModified($article->getPublishedAt());

        // Set response as public. Otherwise it will be private by default.
        $response->setPublic();

        // Check that the Response is not modified for the given Request
        if ($response->isNotModified($request)) {
            // return the 304 Response immediately
            return $response;
        }

        // do more work here - like retrieving more data
        $comments = ...;

        // or render a template with the $response you've already started
        return $this->render('article/show.html.twig', [
            'article' => $article,
            'comments' => $comments,
        ], $response);
    }
}
```
* When the Response is not modified, the isNotModified() automatically sets the response status code to 304, removes the content, and removes some headers that must not be present for 304 responses.

### Last-Modified

* Indicates date and time of last modification
* Cache sets an `If-Modified-Since` header on the request (whose value will be the `Last-Modified` of the original cached response) before sending the request to the app.
```php
// src/Controller/ArticleController.php
namespace App\Controller;

// ...
use App\Entity\Article;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

class ArticleController extends AbstractController
{
    public function show(Article $article, Request $request): Response
    {
        $author = $article->getAuthor();

        $articleDate = new \DateTime($article->getUpdatedAt());
        $authorDate = new \DateTime($author->getUpdatedAt());

        $date = $authorDate > $articleDate ? $authorDate : $articleDate;

        $response = new Response();
        $response->setLastModified($date);
        // Set response as public. Otherwise it will be private by default.
        $response->setPublic();

        if ($response->isNotModified($request)) {
            return $response;
        }

        // ... do more work to populate the response with the full content

        return $response;
    }
}
```

## Client Side Caching

## Server Side Caching

## Edge Side Includes

* Gateway caches can only cache a whole page, so we can use edge side includes to embed content inside your page with its own cache rules
* When all the ESI tags have been resolved, the gateway cache merges each into the main page and sends the final content to the client.
* ESI specification (wrote by Akamai in 2001) describes tags that can be embedded in the page to communicate with the gateway cache. In Symfony, only the `include` tag is available
```html
<body>
        <!-- ... some content -->

        <!-- Embed the content of another page here -->
        <esi:include src="http://..."/>

        <!-- ... more content -->
    </body>
```
* each ESI tags requires a fully qualified URL

### Usage in Symfony

```yaml
# config/packages/framework.yaml
framework:
    # ...
    esi: true
```
```twig
{# templates/static/about.html.twig #}

{# you can use a controller reference #}
{{ render_esi(controller('App\\Controller\\NewsController::latest', { 'maxPerPage': 5 })) }}

{# ... or a URL #}
{{ render_esi(url('latest_news', { 'maxPerPage': 5 })) }}
```

* using the `render_esi()` function rather than the html tag makes your application work even if there is no gateway cache installed.
* the maxPerPage variable you pass is available as an argument to your controller (i.e. $maxPerPage). The variables passed through render_esi also become part of the cache key so that you have unique caches for each combination of variables and values.
* Symfony considers that a gateway cache supports ESI if its request include the Surrogate-Capability HTTP header and the value of that header contains the ESI/1.0 string anywhere.
* when using a controller reference as parameter of `render_esi()`, Symfony has to generate a unique URL. FragmentListener must be enabled : 
```yaml
# config/packages/framework.yaml
framework:
    # ...
    fragments: { path: /_fragment }
```
* Available options for `render_esi()` :
    - `alt`
    Used as the alt attribute on the ESI tag, which allows you to specify an alternative URL to be used if the src cannot be found.
    - `ignore_errors`
    If set to true, an onerror attribute will be added to the ESI with a value of continue indicating that, in the event of a failure, the gateway cache will remove the ESI tag silently.
    - `absolute_uri`
    If set to true, an absolute URI will be generated. default: false

## Available Options For Cache Attribute and setCache() method

```php
// use this method to set several cache settings in one call
// (this example lists all the available cache settings)
$response->setCache([
    'must_revalidate'  => false,
    'no_cache'         => false,
    'no_store'         => false,
    'no_transform'     => false,
    'public'           => true,
    'private'          => false,
    'proxy_revalidate' => false,
    'max_age'          => 600,
    's_maxage'         => 600,
    'immutable'        => true,
    'last_modified'    => new \DateTime(),
    'etag'             => 'abcdef',
    'stale_if_error' => true,         // RFC5861
    'stale_while_revalidate' => true, // RFC5861
]);
```
