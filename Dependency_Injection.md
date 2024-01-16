# Dependency Injection

Dependency Injection is the name of the method of adding dependencies to your __construct() method.

## Service Container

* The service container is an object where other services "live inside"
* To see which services ara available run `bin/console debug:autowiring`
* You can limit services to a specific environment using the `#[When]` attributes (that you can use multiple times on the top of the same class) :
```php
#[When(env: 'dev')]
class SomeClass
{
    // ...
}
```
* You can inject specific values or services in your configuration (`services.arguments` in services.yaml) and you can run these commands to check :
For a list of possible logger services that can be used with autowiring, run:
`bin/console debug:autowiring logger`
For a full list of all possible services in the container, run:
`bin/console debug:container`
* You can add a callable to your constructor :
```php
// src/Service/MessageGenerator.php
namespace App\Service;

use Psr\Log\LoggerInterface;

class MessageGenerator
{
    private string $messageHash;

    public function __construct(
        private LoggerInterface $logger,
        callable $generateMessageHash,
    ) {
        $this->messageHash = $generateMessageHash();
    }
    // ...
}
```
```yaml
# config/services.yaml
services:
    # ... same code as before

    # explicitly configure the service
    App\Service\MessageGenerator:
        arguments:
            $logger: '@monolog.logger.request'
            $generateMessageHash: !closure '@App\Hash\MessageHashGenerator'
```
where MessageHashGenerator has an `__invoke` method.
* You can use the `bind` keyword in your services.yaml file (under the `_defaults` key) to define any arguments used in any services
* The `autoconfigure: true` option tags your services by default. It also works with attributes like `#[AsEventListener]` , `#[AsCommand]` or `#[AsMessageHandler]`
* The `lint:container` command checks that the arguments injected into services match their type declarations (hurts performance a lot).
* **Service are defined private by default** so they can't be accessed with `$container->get()` (bad parctice. It's better calling them with dependency injection). You can define a service as public with the `public: true` option in your services.yaml file.
* The id of each service is its fully-qualified class name.
* To add multiple service configs under the same namespace, you have to use the `namespace` option like this :
```yaml
# config/services.yaml
services:
    command_handlers:
        namespace: App\Domain\
        resource: '../src/Domain/*/CommandHandler'
        tags: [command_handler]

    event_subscribers:
        namespace: App\Domain\
        resource: '../src/Domain/*/EventSubscriber'
        tags: [event_subscriber]
```
* If you have a functional interface (interface with a single method) and a service that defines several methods and one of them with the same name as the one in the functional interface, you can use the `#[AutowireCallable]` attribute when injecting the interface.

Functional interface :
```php
// src/Service/MessageFormatterInterface.php
namespace App\Service;

interface MessageFormatterInterface
{
    public function format(string $message, array $parameters): string;
}
```
Service class :
```php
// src/Service/MessageFormatterInterface.php
namespace App\Service;

class MessageUtils
{
    // other methods...

    public function format(string $message, array $parameters): string
    {
        // ...
    }
}
```
Autowiring :
```php
namespace App\Service\Mail;

use App\Service\MessageFormatterInterface;
use App\Service\MessageUtils;
use Symfony\Component\DependencyInjection\Attribute\AutowireCallable;

class Mailer
{
    public function __construct(
        #[AutowireCallable(service: MessageUtils::class, method: 'format')]
        private MessageFormatterInterface $formatter
    ) {
    }

    public function sendMail(string $message, array $parameters): string
    {
        $formattedMessage = $this->formatter->format($message, $parameters);

        // ...
    }
}
```

## Built-in services

A full list can be accessed with the command `debug:container` (Router, Console, Cache, argument_resolver etc)

## Service Decoration

* When a service is decorated (i.e. with #[AsDecorator(decorates: OldService::class)]) a reference of the old one is kept as `.inner`
* Using the `#[AsDecorator]` attribute, you can access the decorated service by injecting it but in some cases you will need to inject it explicitly with the `#[AutowiredDecorator]` attrubute on top of the argument.
* You can set a priority option in the #[AsDecorator] attribute (higher applied first). Alternatively, you can create a stack of oredered services, each one decorating the next :
```yaml
# config/services.yaml
services:
    # using the short syntax:
    decorated_foo_stack:
        stack:
            - Baz: ['@.inner']
            - Bar: ['@.inner']
            - Foo: ~

    # can be simplified when autowiring is enabled:
    decorated_foo_stack:
        stack:
            - Baz: ~
            - Bar: ~
            - Foo: ~
```
is equivalent to :
```php
$this->services['decorated_foo_stack'] = new Baz(new Bar(new Foo()));
```

## Tags

* Service tags are a way to tell Symfony or other third-party bundles that your service should be registered in some special way
To display services tagged in your app :
```console
php bin/console debug:container --tags
```
* `autoconfigure` options allows Symfony to automatically apply some tags
* Here is the way to add tags automatically for our own services :
```yaml
# config/services.yaml
services:
    # this config only applies to the services created by this file
    _instanceof:
        # services whose classes are instances of CustomInterface will be tagged automatically
        App\Security\CustomInterface:
            tags: ['app.custom_tag']
    # ...
```
That could also be done using the #[AutoconfigureTag('app.cusotm_tag')] on the base class or interface.

These two syntaxes are the same : 
```yaml
# config/services.yaml
services:
    # Compact syntax
    MailerSendmailTransport:
        class: \MailerSendmailTransport
        tags: ['app.mail_transport']

    # Verbose syntax
    MailerSendmailTransport:
        class: \MailerSendmailTransport
        tags:
            - { name: 'app.mail_transport' }
```
* To inject all services tagged with a given one, you can use the #[TaggedIterator] attribute (which also hs an exclude option) :
```php
// src/HandlerCollection.php
namespace App;

use Symfony\Component\DependencyInjection\Attribute\TaggedIterator;

class HandlerCollection
{
    public function __construct(
        // the attribute must be applied directly to the argument to autowire
        #[TaggedIterator('app.handler')]
        iterable $handlers
    ) {
    }
}
```
(it may cause an error with some IDEs when using constructor promotion. That can safely be ignored)
* Tags can have a priority option (which can also be directly defined using the `getDefaultPriority` static method on the service itself)
* You can configure both priority and the index with attribute #[AsTaggedItem] directly on the class of the service : 
```php
// src/Handler/One.php
namespace App\Handler;

use Symfony\Component\DependencyInjection\Attribute\AsTaggedItem;

#[AsTaggedItem(index: 'handler_one', priority: 10)]
class One
{
    // ...
}
```

## Compiler passes

* The Dependency Injection component comes wit some built-in compiler passes which are registered for compilation (when `$container->compile();` is running) like `CheckDefinitionValidityPass`.
* You can add your own passes, by implementing CompilerPassInterface and writing your logic inside the `process()` method : 
```php
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class CustomPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        // ... do something during the compilation
    }
}
```
Then you can add your custom pass to the container inside the `build()` method of the Kernel class : 
```php
// src/Kernel.php
namespace App;

use App\DependencyInjection\Compiler\CustomPass;
use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel
{
    use MicroKernelTrait;

    // ...

    protected function build(ContainerBuilder $container): void
    {
        $container->addCompilerPass(new CustomPass());
    }
}
```
* If you just want to work with tagged services, which is the most common use case of compiler passes, you can also directly implements CompilerPassInterface with the Kernel class :
```php
// src/Kernel.php
namespace App;

use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel implements CompilerPassInterface
{
    use MicroKernelTrait;

    // ...

    public function process(ContainerBuilder $container): void
    {
        // in this method you can manipulate the service container:
        // for example, changing some container service:
        $container->getDefinition('app.some_private_service')->setPublic(true);

        // or processing tagged services:
        foreach ($container->findTaggedServiceIds('some_tag') as $id => $tags) {
            // ...
        }
    }
}
```

## Services Autowiring

* Autowiring allows to pass automatically the right service by reading your type-hinted arguments in the constructor. (It throws an exception if which dependency to inject is not clearly defined)
* The type-hint exactly matches the id because by default service id is its FQCN. So if the service id is not its FQCN, you have to use an alias :
```yaml
# config/services.yaml
services:
    # ...

    # the id is not a class, so it won't be used for autowiring
    app.rot13.transformer:
        class: App\Util\Rot13Transformer
        # ...

    # but this fixes it!
    # the "app.rot13.transformer" service will be injected when
    # an App\Util\Rot13Transformer type-hint is detected
    App\Util\Rot13Transformer: '@app.rot13.transformer'
```
* To useautowiring when type hinting an interface, you'll have to create an alias (unless if only one service implements the type-hinted interface) : 
```yaml
# config/services.yaml
services:
    # ...

    App\Util\Rot13Transformer: ~

    # the ``App\Util\Rot13Transformer`` service will be injected when
    # an ``App\Util\TransformerInterface`` type-hint is detected
    App\Util\TransformerInterface: '@App\Util\Rot13Transformer'
```
* If you have two services implementing the same interface, you can target a specific service with an argument name : 
```yaml
# config/services.yaml
services:
    # ...

    App\Util\Rot13Transformer: ~
    App\Util\UppercaseTransformer: ~

    # the ``App\Util\UppercaseTransformer`` service will be
    # injected when an ``App\Util\TransformerInterface``
    # type-hint for a ``$shoutyTransformer`` argument is detected.
    App\Util\TransformerInterface $shoutyTransformer: '@App\Util\UppercaseTransformer'

    # If the argument used for injection does not match, but the
    # type-hint still matches, the ``App\Util\Rot13Transformer``
    # service will be injected.
    App\Util\TransformerInterface: '@App\Util\Rot13Transformer'
```
* You can also use the #[Target] attribute to specify exactly which service you want to inject : 
```php
// src/Service/MastodonClient.php
namespace App\Service;

use App\Util\TransformerInterface;
use Symfony\Component\DependencyInjection\Attribute\Target;

class MastodonClient
{
    public function __construct(
        #[Target('app.uppercase_transformer')]
        private TransformerInterface $transformer
    ){
    }
}
```
* you can also use the #[Autowire] parameter attribute to instruct autowiring logic (even when argument is not an object) :
```php
// src/Service/MessageGenerator.php
namespace App\Service;

use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

class MessageGenerator
{
    public function __construct(
        // for a service
        #[Autowire(service: 'monolog.logger.request')]
        LoggerInterface $logger,
        
        // use the %...% syntax for parameters
        #[Autowire('%kernel.project_dir%/data')]
        string $dataDir,

        // or use argument "param"
        #[Autowire(param: 'kernel.debug')]
        bool $debugMode,

        // expressions
        #[Autowire(expression: 'service("App\\\Mail\\\MailerConfiguration").getMailerMethod()')]
        string $mailerMethod

        // environment variables
        #[Autowire(env: 'SOME_ENV_VAR')]
        string $senderName
    ) {
    }
    // ...
}
```
[See complex expressions](https://symfony.com/doc/current/service_container/expression_language.html)

## Service Locators

For purpose of lazy loading (A lazy ghost object is an object that is created empty and that is able to initialize itself when being accessed for the first time), Service Subscribers gives access to a set of predefined services while instantiating them only when actually needed through a Service Locator, a separate lazy-loaded container.

* To include services, you 'll have to implement [ServiceSubscriberInterface](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Contracts/Service/ServiceSubscriberInterface.php) and use static `getSubscribedServices()`
* You can define directly the locator by injecting an instance of ContainerInterface with the #[AutowireLocator] attribute : 
```php
// src/CommandBus.php
namespace App;

use App\CommandHandler\BarHandler;
use App\CommandHandler\FooHandler;
use Psr\Container\ContainerInterface;
use Symfony\Component\DependencyInjection\Attribute\AutowireLocator;

class CommandBus
{
    public function __construct(
        #[AutowireLocator([
            FooHandler::class,
            BarHandler::class,
        ])]
        private ContainerInterface $handlers,
    ) {
    }

    public function handle(Command $command): mixed
    {
        $commandClass = get_class($command);

        if ($this->handlers->has($commandClass)) {
            $handler = $this->handlers->get($commandClass);

            return $handler->handle($command);
        }
    }
}
```
* You can also use the #[TaggedLocator] which will reference all the services with the tag passed as an argument. (And then you have access to all the services of the service locator) with `$this->locator->get('App\FooCommand')` : 
```php
// src/CommandBus.php
namespace App;

use Psr\Container\ContainerInterface;
use Symfony\Component\DependencyInjection\Attribute\TaggedLocator;

class CommandBus
{
    public function __construct(
        // creates a service locator with all the services tagged with 'app.handler'
        #[TaggedLocator('app.handler')]
        private ContainerInterface $locator,
    ) {
    }
}
```
* To reuse a service locator : 
```yaml
# config/services.yaml
services:
    app.command_handler_locator:
        class: Symfony\Component\DependencyInjection\ServiceLocator
        arguments:
            -
                App\FooCommand: '@app.command_handler.foo'
                App\BarCommand: '@app.command_handler.bar'
        # if you are not using the default service autoconfiguration,
        # add the following tag to the service definition:
        # tags: ['container.service_locator']
```
and then :
```php
// src/CommandBus.php
namespace App;

use Psr\Container\ContainerInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

class CommandBus
{
    public function __construct(
        #[Autowire(service: 'app.command_handler_locator')]
        private ContainerInterface $locator,
    ) {
    }
}
```
* To use a service locator in a compiler pass, use the `ServiceLocatorTagPass::register($container, $locateableServices)` method.