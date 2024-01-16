# Automated Tests

## Unit tests with PHPUnit

Nothing particular in a Symfony context.
* By convention, the tests/ directory should replicate the directory of our app for unit tests
* autoloading is automatically enabled via the default config in `phpunit.xml.dist` provided by Symfony flex.

## Functional tests with PHPUnit

* We can extend KernelTestCase that can create and boot Symfony kernel in our app. It also assures that kernel is rebooted for each test.
* The tests create a kernel that runs in the `test` environment. This allows to have special settings for your tests inside `config/packages/test/`.
*  It is recommended to run our tests with `debug` mode set to `false` in the CI server to improve perfs (and manually clear the cache when necessary).
*  Same priority as usual for .env files (last read file is .env.test.local in `test` environment) but .env.local is never used in test environment
* To fetch a service in your test :
```php
// tests/Service/NewsletterGeneratorTest.php
namespace App\Tests\Service;

use App\Service\NewsletterGenerator;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class NewsletterGeneratorTest extends KernelTestCase
{
    public function testSomething(): void
    {
        // (1) boot the Symfony kernel
        self::bootKernel();

        // (2) use static::getContainer() to access the service container
        $container = static::getContainer();

        // (3) run some service & test the result
        $newsletterGenerator = $container->get(NewsletterGenerator::class);
        $newsletter = $newsletterGenerator->generateMonthlyNews(/* ... */);

        $this->assertEquals('...', $newsletter->getContent());
    }
}
```
* Use the [DAMADoctrineTestBundle](https://github.com/dmaicher/doctrine-test-bundle) to reset databases betwenn each test.

## Client object

Making your test class extending WebTestCase, you'll be able to create a client and make requests.

* Theclient object simulates a browser behaviour and supports many operations
```php
$client->back();
$client->forward();
$client->reload();

// clears all cookies and the history
$client->restart();
```
* The client does not follow automatically redirect Responses, you have to use the<br> `$crawler = $client->followRedirect()` method or `$client->followRedirects()` if you want to force them for all redirections. Passing `$client->followRedirects(false)` will stop this behaviour.
* You can login a user in your test :
```php
// tests/Controller/ProfileControllerTest.php
namespace App\Tests\Controller;

use App\Repository\UserRepository;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class ProfileControllerTest extends WebTestCase
{
    // ...

    public function testVisitingWhileLoggedIn(): void
    {
        $client = static::createClient();
        $userRepository = static::getContainer()->get(UserRepository::class);

        // retrieve the test user
        $testUser = $userRepository->findOneByEmail('john.doe@example.com');

        // simulate $testUser being logged in
        $client->loginUser($testUser);

        // test e.g. the profile page
        $client->request('GET', '/profile');
        $this->assertResponseIsSuccessful();
        $this->assertSelectorTextContains('h1', 'Hello John!');
    }
}
```
* The `loginUser()` method creates a special TestBrowserToken object and stores in the session of the test client. If you need to define custom attributes in this token, you can use the tokenAttributes argument of the loginUser() method. This method does not work using stateless firewalls.
* You can make AJAX requests using `xmlHttpRequest()` method.

### Make multiple requests

The kernel is rebooted after making a request and recreates the container from scratch. 
* The client's `disableReboot()` method will reset the Kernel by calling the reset() methods on all services tagged with `kernel.reset`
* If you do not want some services to be resetted, you'll have to use a compiler pass :
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

    protected function process(ContainerBuilder $container): void
    {
        if ('test' === $this->environment) {
            // prevents the security token to be cleared
            $container->getDefinition('security.token_storage')->clearTag('kernel.reset');

            // prevents Doctrine entities to be detached
            $container->getDefinition('doctrine')->clearTag('kernel.reset');

            // ...
        }
    }
}
```

## Crawler object

* The crawler object is returned by the client's `request()` methods whose signature is :
```php
request(
    string $method,
    string $uri,
    array $parameters = [],
    array $files = [],
    array $server = [],
    string $content = null,
    bool $changeHistory = true
): Crawler
```
* This object allows to perform multiple actions on html and xml doncuments.
    - `filter('h1.title')`: Nodes that match the CSS selector.
    - `filterXpath('h1')`: Nodes that match the XPath expression.
    - `eq(1)`: Node for the specified index.
    - `first()`: First node.
    - `last()`: Last node.
    - `siblings()`: Siblings.
    - `nextAll()`: All following siblings.
    - `previousAll()`: All preceding siblings.
    - `parents()`: Returns the parent nodes.
    - `children()`: Returns children nodes.
    - `reduce($lambda)`: Nodes for which the callable does not return false.
    - `$crawler->selectLink('Click Here')->link()`: gives access to the [Link](https://github.com/symfony/symfony/blob/7.0/src/Symfony/Component/DomCrawler/Link.php) object
* It can also extract informations.
```php
// returns the attribute value for the first node
$crawler->attr('class');

// returns the node value for the first node
$crawler->text();

// returns the default text if the node does not exist
$crawler->text('Default text content');

// pass TRUE as the second argument of text() to remove all extra white spaces, including
// the internal ones (e.g. "  foo\n  bar    baz \n " is returned as "foo bar baz")
$crawler->text(null, true);

// extracts an array of attributes for all nodes
// (_text returns the node value)
// returns an array for each element in crawler,
// each with the value and href
$info = $crawler->extract(['_text', 'href']);

// executes a lambda for each node and return an array of results
$data = $crawler->each(function ($node, int $i): string {
    return $node->attr('href');
});
```

## Profiler object

You can enable the Symfony profiler during tests. As it slows down your tests, it is disabled by default
```yaml
# config/packages/test/web_profiler.yaml

# ...
framework:
    profiler: { enabled: true, collect: false }
```
Setting `collect` to true enables profiler for all tests. To enable in a specific test, use the `$client->enableProfiler()` method.
```php
// tests/Controller/LuckyControllerTest.php
namespace App\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class LuckyControllerTest extends WebTestCase
{
    public function testRandomNumber(): void
    {
        $client = static::createClient();

        // enable the profiler only for the next request (if you make
        // new requests, you must call this method again)
        // (it does nothing if the profiler is not available)
        $client->enableProfiler();

        $crawler = $client->request('GET', '/lucky/number');

        // ... write some assertions about the Response

        // check that the profiler is enabled
        if ($profile = $client->getProfile()) {
            // check the number of requests
            $this->assertLessThan(
                10,
                $profile->getCollector('db')->getQueryCount()
            );

            // check the time spent in the framework
            $this->assertLessThan(
                500,
                $profile->getCollector('time')->getDuration()
            );
        }
    }
}
```

## Framework objects access

To retrieve services, we need to get access to the container :
```php
// tests/Service/NewsletterGeneratorTest.php
namespace App\Tests\Service;

use App\Service\NewsletterGenerator;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class NewsletterGeneratorTest extends KernelTestCase
{
    public function testSomething(): void
    {
        // (1) boot the Symfony kernel
        self::bootKernel();

        // (2) use static::getContainer() to access the service container
        $container = static::getContainer();

        // (3) run some service & test the result
        $newsletterGenerator = $container->get(NewsletterGenerator::class);
        $newsletter = $newsletterGenerator->generateMonthlyNews(/* ... */);

        $this->assertEquals('...', $newsletter->getContent());
    }
}
```

## Client configuration

* You can configure your test env from the `bootKernel()` method.
```php
self::bootKernel([
    'environment' => 'my_test_env',
    'debug'       => false,
]);
```
* You can also specify config in the `createClient()` method whose signature is
```php
protected static function createClient(array $options = [], array $server = []): KernelBrowser
```
e.g.
```php
$client = static::createClient([], [
    'HTTP_HOST'       => 'en.example.com',
    'HTTP_USER_AGENT' => 'MySuperBrowser/1.0',
]);
```

## Request and response objects introspection

Here are the ways to access Request and Response and other internals object :
```php
$history = $client->getHistory();
$cookieJar = $client->getCookieJar();

// the HttpKernel request instance
$request = $client->getRequest();

// the BrowserKit request instance
$request = $client->getInternalRequest();

// the HttpKernel response instance
$response = $client->getResponse();

// the BrowserKit response instance
$response = $client->getInternalResponse();

// the Crawler instance
$crawler = $client->getCrawler();
```
See [some assertions](https://symfony.com/doc/current/testing.html#testing-the-response-assertions) here.

## PHPUnit bridge

The PHPUnit Bridge provides utilities to report legacy tests and usage of deprecated code and helpers for mocking native functions related to time, DNS and class existence.

It comes with the following features:

* Sets by default a consistent locale (C) for your tests (if you create locale-sensitive tests, use PHPUnit's setLocale() method);
* Auto-register class_exists to load Doctrine annotations (when used);
* It displays the whole list of deprecated features used in the application;
* Displays the stack trace of a deprecation on-demand;
Provides a ClockMock, DnsMock and ClassExistsMock classes for tests sensitive to time, network or class existence;
* Provides a modified version of PHPUnit that allows:
    - separating the dependencies of your app from those of phpunit to prevent any unwanted constraints to apply;
    - running tests in parallel when a test suite is split in several phpunit.xml files;
    - recording and replaying skipped tests;
* It allows to create tests that are compatible with multiple PHPUnit versions (because it provides polyfills for missing methods, namespaced aliases for non-namespaced classes, etc.).


[See docs](https://symfony.com/doc/current/components/phpunit_bridge.html)

## Handling legacy deprecated code

* In case you need to inspect the stack trace of a particular deprecation triggered by your unit tests, you can set the `SYMFONY_DEPRECATIONS_HELPER` environment variable to a regular expression that matches this deprecation's message, enclosed with /.
```xml
<!-- https://phpunit.de/manual/6.0/en/appendixes.configuration.html -->
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/6.0/phpunit.xsd"
>

    <!-- ... -->

    <php>
        <server name="KERNEL_CLASS" value="App\Kernel"/>
        <env name="SYMFONY_DEPRECATIONS_HELPER" value="/foobar/"/>
    </php>
</phpunit>
```

[More details in the docs](https://symfony.com/doc/current/components/phpunit_bridge.html#configuration)
