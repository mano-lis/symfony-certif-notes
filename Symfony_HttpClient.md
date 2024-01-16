# Symfony HttpClient Component


## Basic Usage

Autowire [HttpClient](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/HttpClient/HttpClient.php) (type-hinting HttpClientInterface) to build a Request (with request() method) and return a Response 

Most important methods attached to Response object :

```php
$statusCode = $response->getStatusCode();
// $statusCode = 200
$contentType = $response->getHeaders()['content-type'][0];
// $contentType = 'application/json'
$content = $response->getContent();
// $content = '{"id":521583, "name":"symfony-docs", ...}'
$content = $response->toArray();
// $content = ['id' => 521583, 'name' => 'symfony-docs', ...]
```

## Configuration

Use the `default_options` key to configure globally the way requests are performed or the `withOptions()` method from HttpClient service to override global config.
-> max_host_connections can't be overriden by a request
-> [see all options](https://symfony.com/doc/current/reference/configuration/framework.html#reference-http-client)

### Scoped Client

* to scope client with a specific config you can use the `scope` option (in your yaml file) with a regular expression (or `base_uri` for your main client)
* Each scoped client must have a specific service. A correpsonding named autowiring alias is defined (if the key of the scoped client in framework.yaml is `github.client`, corresponding service will be autowired with `Symfony\Contracts\HttpClient\HttpClientInterface $githubClient`)

## Making Requests

One single method `request()` to perform all kind of https requests. Calls to this method returns immediately instead of waiting to receive the response.

### Authentication 

Can be configured globally and to each request

### Query String Parameters

can be defined directly in the url or in an associative array in parameters of the `request()` method

### Headers

same for authentication

### Uplodaing Data

* Several methods can be used with the `body` option (in the option parameter of the `request()` method) such as closures, regular strings, iterables...
* If `Content-Type` header is not defined explicitly, default will be `'Content-Type: application/x-www-form-urlencoded'`
* Use `json` instead of `body` if you're uploading json (content will be json-encoded automatically)

### Cookies

Http component provides a stateless client so you can use [BrowserKitcomponent](https://symfony.com/doc/current/components/browser_kit.html#component-browserkit-sending-cookies) or set it manually 

```php
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\HttpFoundation\Cookie;

$client = HttpClient::create([
    'headers' => [
        'Cookie' => new Cookie('flavor', 'chocolate', strtotime('+1 day')),

        // you can also pass the cookie contents as a string
        'Cookie' => 'flavor=chocolate; expires=Sat, 11 Feb 2023 12:18:13 GMT; Max-Age=86400; path=/'
    ],
]);
```

### Retry Failed Requests

* Configurable with `retry_failed` option
* By default retries three times (exponential delay), only for these status code with any http methods (423, 425, 429, 502 and 503) and these one with idempotent methods (500, 504, 507 and 510)
* It's possible to retry over different base URI by adding an array with `base_uri` key into parameters of request() method

## Processing Responses

Here are the methods provided by the Responses object ([ResponseInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Contracts/HttpClient/ResponseInterface.php))

```php
$response = $client->request('GET', 'https://...');

// gets the HTTP status code of the response
$statusCode = $response->getStatusCode();

// gets the HTTP headers as string[][] with the header names lower-cased
$headers = $response->getHeaders();

// gets the response body as a string
$content = $response->getContent();

// casts the response JSON content to a PHP array
$content = $response->toArray();

// casts the response content to a PHP stream resource
$content = $response->toStream();

// cancels the request/response
$response->cancel();

// returns info coming from the transport layer, such as "response_headers",
// "redirect_count", "start_time", "redirect_url", etc.
$httpInfo = $response->getInfo();

// you can get individual info too
$startTime = $response->getInfo('start_time');
// e.g. this returns the final response URL (resolving redirections if needed)
$url = $response->getInfo('url');

// returns detailed logs about the requests and responses of the HTTP transaction
$httpLogs = $response->getInfo('debug');

// the special "pause_handler" info item is a callable that allows to delay the request
// for a given number of seconds; this allows you to delay retries, throttle streams, etc.
$response->getInfo('pause_handler')(2);
```



