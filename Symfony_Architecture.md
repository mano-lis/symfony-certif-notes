# Symfony Architecture

## Symfony Flex

Used by default since v4, Flex is a composer plugin for Symfony which provides auto install and config files for bundles and components, dependencies management and other features. It also respects a pattern for file organization :
```
your-project/
├── assets/
├── bin/
│   └── console
├── config/
│   ├── bundles.php
│   ├── packages/
│   ├── routes.yaml
│   └── services.yaml
├── public/
│   └── index.php
├── src/
│   ├── ...
│   └── Kernel.php
├── templates/
├── tests/
├── translations/
├── var/
└── vendor/
``` 

## License

Code is licensed under the [MIT License](https://en.wikipedia.org/wiki/MIT_License) and documentation under Creative Commons ([CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/))

## Components

Symfony Components are decoupled libraries for PHP applications. They can be used inside a Symfony application or in a stand-alone way.

## Bridges

Bridge are set of classes that aims at extending a library into Symfony. the most known are Twig, PHPUnit, Doctrine or Monolog.

## Code organization

See example above in Symfony Flex section

## Request handling

Each event of the symfony kernel is a subclass of [KernelEvent](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/HttpKernel/Event/KernelEvent.php) which provides these methods :
* getRequestType()
Returns the type of the request (HttpKernelInterface::MAIN_REQUEST or HttpKernelInterface::SUB_REQUEST).
* getKernel()
Returns the Kernel handling the request.
* getRequest()
Returns the current Request being handled.
* isMainRequest()
Checks if this is a main request.

Here are the lists of the events, in order of appearance :

* **kernel.request**
  dispatched very early, before controller is determined.
* **kernel.controller**
  dispatched after constroller was resolved, but before its execution
* **kernel.controller_arguments**
  just before controller is called
* **kernel.view**
  dispatched after controller execution, only if the controller does not return a Response object
* **kernel.response**
  just after kernel.view or just after controller returns a Response object
* **kernel.finish_request**
* **kernel.terminate**
  after response has been sent
* **kernel.exception**
  as soon as an error occurs during the handling of an http request

## Exception handling

In Symfony applications, **all errors are treated as exceptions**. 
In dev env you got the typical exception page, and in prod env there are 4 ways to customize error pages:
1. If you only want to change the contents and styles of the error pages to match the rest of your application, override the default error templates: 
   errorXXX.html.twig in templates/bundles/TwigBundle/Exception/, error.html.twig will handle all html errors which won't have a specific template.
2. If you want to change the contents of non-HTML error output, create a new normalizer;
3. If you also want to tweak the logic used by Symfony to generate error pages, override the default error controller:
   This controller can be used in dev using `_error` prefix (http://localhost/_error/{statusCode}) but if you need to completely override it, specify your own controller method like this:
    ```yaml
    # config/packages/framework.yaml
    framework:
        error_controller: App\Controller\ErrorController::show
    ```
4. If you need total control of exception handling to run your own logic use the kernel.exception event and write your own EventListener
   
## Event dispatcher and kernel events

* Event Dispatcher implements the [Mediator](https://en.wikipedia.org/wiki/Mediator_pattern) and [Observer](https://en.wikipedia.org/wiki/Observer_pattern) design patterns.
* An Event has to be an Symfony/Contracts/EventDispatcher/Event instance to be dispatched and have a unique name
* event propagation can be stopped by calling stopPropagation() method on the event

## Official best practices

### Creating the Project

* **Use the symfony binary to create applications**
* **Use default directory structure**

### Configuration

* **Use Environment Variables for Infrastructure Configuration**
* **Use secrets for sensitive informations**
* **Use Parameters for Application Configuration** (defines in service.yaml)
    ```yaml
    # config/services.yaml
    parameters:
        # the parameter name is an arbitrary string (the 'app.' prefix is recommended
        # to better differentiate your parameters from Symfony parameters).
        app.admin_email: 'something@example.com'
    ```
    (can be reused with special syntax '%app.admin_email%')

* **Use Short and Prefixed Parameter Names**

### Business Logic

* **Services Should be Private Whenever Possible**

### Templates

* **Use Snake Case for Template Names and Variables**
* **Prefix Template Fragments with an Underscore**

### Forms

* **Add Form Buttons in Templates**
* **Define Validation Constraints on the Underlying Object**
* **Use a Single Action to Render and Process the Form**

### Internationalization

* **Use the XLIFF Format for Your Translation Files**
* **Use Keys for Translations Instead of Content Strings**

### Security

* **Define a Single Firewall**
* **Use the auto Password Hasher**
* **Use Voters to Implement Fine-grained Security Restrictions**

### Testing

**Smoke Test your URLs**
**Hard-code URLs in a Functional Test**

## Release management

* vX.4 are LTS
* vX.4 and v(X+1).0 are equivalent except deprecation from vX.4 won't work in v(X+1).0
* major version released each two years in November
* minor version ~6months

## Backward compatibility promise

No BC break in patch and minor versions

## Deprecations best practices

* Annotate a deprecated method with the version where deprecation was introduced
* The deprecation must be added to the CHANGELOG.md file of the impacted or the UPGRADE.md file for a minor version
* When removing deprecated code, the consequences of the deprecation must be added to the CHANGELOG.md file of the impacted component:

## Framework overloading

?

## Release management and roadmap schedule

see above

## Framework interoperability and PSRs

## Naming conventions

See [Naming Conevntions and all Coding Standards](https://symfony.com/doc/current/contributing/code/standards.html)
