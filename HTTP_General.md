# HTTP

## Verbs

* GET - The GET method requests a representation of the specified resource. Requests using GET should only retrieve data.
* POST - The POST method submits an entity to the specified resource, often causing a change in state or side effects on the server.
* PUT - The PUT method replaces all current representations of the target resource with the request payload.
* PATCH - The PATCH method applies partial modifications to a resource.
* DELETE - The DELETE method deletes the specified resource.
* HEAD - The HEAD method asks for a response identical to a GET request, but without the response body.
* TRACE - The TRACE method performs a message loop-back test along the path to the target resource.
 (Often disabled for security reasons)
* CONNECT - The CONNECT method establishes a tunnel to the server identified by the target resource. (Often used with https protocol to establish secure connexion via proxy)
* OPTIONS - The OPTIONS method describes the communication options for the target resource (authorized methods, accepted headers..), often used during Cors pre requests context.

## Codes

### Responses codes

1. Informational responses (100 – 199)
2. Successful responses (200 – 299)
3. Redirection messages (300 – 399)
4. Client error responses (400 – 499)
5. Server error responses (500 – 599)

### Some important status codes

#### Success

* 200 OK
* 201 Created
* 202 Accepted (no way for http to send asynchronous response later, often used when another process or server handles the request)
* 204 No-Content

#### Redirection

* 300 Multiple Choices (more than one possible response)
* 301 Moved Permanently (new url given in the repsonse)
* 302 Found (temporarly changed requested resource uri. same uri will therefore be used in future requests)
* 303 See Other (direct the client to get the response at another uri)
* 304 Not Modified (used for caching purposes)
* 307 Temporary Redirect (same as 302, except that user agent must not change HTTP method used in the prior request)
* 308 Permanent Redirect (same as 301, except that user agent must not change HTTP method used in the prior request)

#### Client Error

* 400 Bad Request
* 401 Unauthorized (here means unauthenticated)
* 402 Payment Required
* 403 Forbidden (authenticated but access denied for this user)
* 404 Not Found (api side it could mean that endpoint is valid but resource does not exist. 403 can be sent to hide existence of a resource to the user).
* 405 Method Not Allowed
* 408 Request Timeout
* 415 Unsupported Media Type
* 422 Unprocessable Content
* 429 Too Many Requests

#### Server Error

* 500 Interanl Server Error
* 501 Not Implemented (Requested Method not supported by the server)
* 502 Bad Getaway (server, while working as a gateway got an invalid response)
* 503 Service Unavailable (server not ready, could be used temporarly i.e. when server is down for maintenance)
* 504 Gateway Timeout
* 505 HTTP version not supported

## Headers

### Overview

Headers represent additional informations with an HTTP request or response. They could be grouped according to their contexts :

1. **Request Headers** : Contain more information about the resource to be fetched, or about the client requesting the resource.
2. **Response Headers** : Hold additional information about the response, like its location or about the server providing it.
3. **Representation headers** : Contain information about the body of the resource, like its MIME type, or encoding/compression applied.
4. **Payload Headers** : Contain representation-independent information about payload data, including content length and the encoding used for transport.

- End-to-end headers are headers that must be transmitted to the final recipient of the message (proxies must transmit them unmodified)
- Hop-by-hop headers are meaningful for single transport-level connections

### Authentication 

* [WWW-Authenticate](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/WWW-Authenticate) : Define Authentication method to access a resource
* [Authorization](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization): Contains credentials to authenticate UA with a server

### Caching

* [Age](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Age) : Time (seconds) that the object has been in a proxy cache
* [Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) : Directives for caching mechanisms in both requests and responses. Tricky options with no-cache and no-store. no-cache means informations must be revalidate by the server when requests is made. no-store means that the resource can't be cached.
* [Expires](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Expires) : Date/time after which response is considered expired

### Conditionals

* [Last-Modified](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified)
* [ETag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag) : Unique string identifying the version of the resource. Used by [If-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Match) et [If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match)
* [If-Modified-Since](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Modified-Since)
* [If-Unmodified-Since](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Unmodified-Since)
* [Vary](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary) : Describes what parts of the message (other than method and url) influenced response content. Most often uses to create a cache key when content-negotiation is used.

### Content negotiation

* [Accept](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept) : types of data that can be sent back by the server
* [Accept-Encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding) : encoding algorithm that can be used on the resource.
* [Accept-Language](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language)

### Cookies

* [Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cookie)
* [Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie)

### Message body information

* [Content-Length](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Length)
* [Content-Type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type)
* [Content-Encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding)
* [Content-Language](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Language)
* [Content-Location](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Location) : Content-Location is used to specify the exact url of the resource whereas [Location](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Location) is used to indicate the url of a moved resource (3xx code).

N.B. Not sure if the following ones are so important for symfony certification

### CORS

* [Access-Control-Allow-Origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Origin)
* [Access-Control-Allow-Credentials](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials)
* [Access-Control-Allow-Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Headers)
* [Access-Control-Allow-Methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Methods)
* [Access-Control-Expose-Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Expose-Headers)
* [Access-Control-Max-Age](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Max-Age)
* [Access-Control-Request-Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Request-Headers)
* [Access-Control-Request-Method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Request-Method)
* [Origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin)

### Proxies

* [Forwarded](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Forwarded) : contains all the infos specifies by the next three headers
* [X-Forwarded-For](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For) : identifies original ip addresses of the client
* [X-Forwarded-Host](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Host) : identifies original host
* [X-Forwarded-Proto](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Proto) : identifies protocol used by the client
* [Via](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Via) : added by proxies

## Symfony and HTTP Fundamentals

### Requests and Responses in Symfony

#### Symfony Request Object

[Request](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/HttpFoundation/Request.php) class is an abstraction of the http request that provides useful methods and makes a lot of work in the background. 

```php
use Symfony\Component\HttpFoundation\Request;

$request = Request::createFromGlobals();

// the URI being requested (e.g. /about) minus any query parameters
$request->getPathInfo();

// retrieves $_GET and $_POST variables respectively
$request->query->get('id');
$request->request->get('category', 'default category');

// retrieves $_SERVER variables
$request->server->get('HTTP_HOST');

// retrieves an instance of UploadedFile identified by "attachment"
$request->files->get('attachment');

// retrieves a $_COOKIE value
$request->cookies->get('PHPSESSID');

// retrieves an HTTP request header, with normalized, lowercase keys
$request->headers->get('host');
$request->headers->get('content-type');

$request->getMethod();    // e.g. GET, POST, PUT, DELETE or HEAD
$request->getLanguages(); // an array of languages the client accepts
```

#### Symfony Response Object

[Response](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/HttpFoundation/Response.php) is an object oriented interface that allows our to construct the http response.

```php
use Symfony\Component\HttpFoundation\Response;

$response = new Response();

$response->setContent('<html><body><h1>Hello world!</h1></body></html>');
$response->setStatusCode(Response::HTTP_OK);

// sets a HTTP response header
$response->headers->set('Content-Type', 'text/html');

// prints the HTTP headers followed by the content
$response->send();
```

### The Journey from the Request to the Response

#### The Front Controller

Instead of having one php file per page, symfony provides a front controller (i.e. /index.php) which handles every requests (such as /index.php/contact or /index.php/blog) coming into our application. Thanks to rewrite rules, url are cleaned (/index.php won't be needed, you'll just use '/blog').

#### The Symfony Application Flow

The [Routing Component](https://symfony.com/doc/current/routing.html) interprets incoming requests and passes them to php functions (in Controllers) which return Response objects


-> **[Complete headers list](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)**

