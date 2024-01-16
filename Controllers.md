# Symfony Controllers

## The base AbstractController class

To provide some useful methods and mark our controller as a public service with the right tag (controller.service_arguments), and use services directly in method parameters, we can extend the [AbstractController](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Bundle/FrameworkBundle/Controller/AbstractController.php) class. 
Using the #[AsController] attribute would do the same job (without providing the methods from AbstractController).

The methods from AbstractController are :
* setContainer()
* getParameter()
* getSubscribedServices()
* generateUrl()
* forward()
* redirect()
* redirectToRoute()
* json()
* file()
* addFlash()
* isGranted()
* denyAccessUnlessGranted()
* renderView()
* renderBlockView()
* render()
* renderBlock()
* stream()
* createNotFoundException()
* createAccessDeniedException()
* createForm()
* createFormBuilder()
* getUser()
* isCsrfTokenValid()
* addLink()
* sendEarlyHints()
* doRender()
* doRenderView()

## Request and Response

Further details in [dedicated file](Http_General.md#symfony-and-http-fundamentals).
* Request object has a request property which is an InputBag object (useful to get the body of your POST request)
* Response has a headers property which is a ResponseInputBag (useful to set headers)
* It's possible to automatically map query parameters or request payload to the controlle's action arguments using attributes (#[MapQueryParameter], #[MapQueryString] or #[MapRequestPayload] with several options).
Let's pretend a user sends you a request with the following query string:<br> `https://example.com/dashboard?firstName=John&lastName=Smith&age=27`.
Here is your DTO class:
```php
namespace App\Model;

use Symfony\Component\Validator\Constraints as Assert;

class UserDTO
{
    public function __construct(
        #[Assert\NotBlank]
        public string $firstName,

        #[Assert\NotBlank]
        public string $lastName,

        #[Assert\GreaterThan(18)]
        public int $age,
    ) {
    }
}
```
You can map the whole query string
```php
use App\Model\UserDto;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapQueryString;

// ...

public function dashboard(
    #[MapQueryString] UserDTO $userDto
): Response
{
    // ...
}
```
or map each query parameter with optional filter regex:
```php
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;

// ...

public function dashboard(
    #[MapQueryParameter(filter: \FILTER_VALIDATE_REGEXP, options: ['regexp' => '/^\w++$/'])] string $firstName,
    #[MapQueryParameter] string $lastName,
    #[MapQueryParameter(filter: \FILTER_VALIDATE_INT)] int $age,
): Response
{
    // ...
}
```

## Cookies
As Google is going to phase out third-party cookie, Symfony provides support for CHIPS cookie (Cookies Having Independent Partitioned State), so Cookie class has now a boolean flag `partitioned`. See details [here](https://symfony.com/blog/new-in-symfony-6-4-chips-cookies)
```php
use Symfony\Component\HttpFoundation\Cookie;

$cookie = new Cookie('cookie-name', 'cookie-value', '...', partitioned: true);

// or:
$cookie = Cookie::fromString('cookie-name=cookie-value; ...; Partitioned;');

// or:
$cookie = ...
$cookie->withPartitioned();
```

## Sessions

Symfony sessions are designed to replace the usage of the $_SESSION super global and native PHP functions related to manipulating the session
Sessions are available through RequestStack and Request object with getSession() method.

### Session Attributes

* Session attributes are key-value pairs managed with the [AttributeBag](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/HttpFoundation/Session/Attribute/AttributeBag.php) class.
* Sessions are automatically started whenever you read, write or even check for the existence of data in the session.
```php
// stores an attribute for reuse during a later user request
$session->set('attribute-name', 'attribute-value');

// gets an attribute by name
$foo = $session->get('foo');

// the second argument is the value returned when the attribute doesn't exist
$filters = $session->get('filters', []);
```

### Flash Messages

* Flash messages are messages stored in the user's session which vanished when session is closed.
```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
// ...

public function update(Request $request): Response
{
    // ...

    if ($form->isSubmitted() && $form->isValid()) {
        // do some sort of processing

        $this->addFlash(
            'notice',
            'Your changes were saved!'
        );
        // $this->addFlash() is equivalent to $request->getSession()->getFlashBag()->add()

        return $this->redirectToRoute(/* ... */);
    }

    return $this->render(/* ... */);
}
```
* Flash messages are accessible in the twig templates via `app.flashes` which returns an array

### Configuration

* Sessions are configurable through framework.yaml :
```yaml
# config/packages/framework.yaml
framework:
    # Enables session support. Note that the session will ONLY be started if you read or write from it.
    # Remove or comment this section to explicitly disable session support.
    session:
        # ID of the service used for session storage
        # NULL means that Symfony uses PHP default session mechanism
        handler_id: null
        # improves the security of the cookies used for sessions
        cookie_secure: auto
        cookie_samesite: lax
        storage_factory_id: session.storage.factory.native
```
* Session can be managed directly by symfony using 'session.handler.native_file' as handler_id.
* To manage cookie server side, we can use `cookie_lifetime` and `gc_maxlifetime` options for the garbage collection.
* Another option is to check with metadata recorded by Symfony : 
```php
$session->start();
if (time() - $session->getMetadataBag()->getLastUsed() > $maxIdleTime) {
    $session->invalidate();
    throw new SessionExpired(); // redirect to expired session page
}
```
See more details about Session (like how to store in a database) [here](https://symfony.com/doc/current/session.html#configuration)

## HTTP redirects

The redirect() method does not check its destination in any way. If you redirect to a URL provided by end-users, your application may be open to the unvalidated redirects security vulnerability.

```php
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\Response;

// ...
public function index(): RedirectResponse
{
    // redirects to the "homepage" route
    return $this->redirectToRoute('homepage');

    // redirectToRoute is a shortcut for:
    // return new RedirectResponse($this->generateUrl('homepage'));

    // does a permanent HTTP 301 redirect
    return $this->redirectToRoute('homepage', [], 301);
    // if you prefer, you can use PHP constants instead of hardcoded numbers
    return $this->redirectToRoute('homepage', [], Response::HTTP_MOVED_PERMANENTLY);

    // redirect to a route with parameters
    return $this->redirectToRoute('app_lucky_number', ['max' => 10]);

    // redirects to a route and maintains the original query string parameters
    return $this->redirectToRoute('blog_show', $request->query->all());

    // redirects to the current route (e.g. for Post/Redirect/Get pattern):
    return $this->redirectToRoute($request->attributes->get('_route'));

    // redirects externally
    return $this->redirect('http://symfony.com/doc');
}
```

## Internal redirects

```php
public function index($name): Response
{
    $response = $this->forward('App\Controller\OtherController::fancy', [
        'name'  => $name,
        'color' => 'green',
    ]);

    // ... further modify the response or return it directly

    return $response;
}
```

## File Upload

* Create a form type with a FileType field (unmapped)
* Change its name to a safe name
* Set filename in a dedicated property of the entity (not the file)
* Render a file in a Controller via the file() method 
[See details](https://symfony.com/doc/current/controller/upload_file.html)

## Built-in Controllers

* ErrorController
* SecurityController

## Argument Value Resolvers

Dynamically get the value inside an url is possible via the controller thanks to the [ArgumentResolver](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/HttpKernel/Controller/ArgumentResolver.php).
Most known built-in value resolvers :
* BackedEnumValueResolver : if the value in the route is not a valid backing value of the enum, route will lead to a 404
* DateTimeValueResolver : By default any input that can be parsed as a date string by PHP is accepted (DateTimeInterface object is generated with the Clock component, giving full control over date and time value)
* RequestValueResolver
* SessionValueResolver
* UidValueResolver
* UserValueResolver : also possible to use our User class if argument has #[CurrentUser] attribute. Requires SecurityBundle
* EntityValueResolver : Automatically query for an entity and pass it to an argument in the controller

[Full list](https://symfony.com/doc/current/controller/value_resolver.html#built-in-value-resolvers)
It's also possible to create a custom value resolver.

