---
title: Automocking Dependencies
description: A way to speed up your test-driven PHP development through automation...
thumbnail: photo-1546900703-cf06143d1239.jpg
date: 2014-02-25 10:57:45
updated: 2020-11-26 09:36:18
tags:
    - Programming
    - Testing
    - PHP
---

<!-- markdownlint-disable no-inline-html -->
![Image](photo-1546900703-cf06143d1239.jpg)<span class="caption">Image: [Joshua Aragon](https://unsplash.com/@goshua13)</span>
<!-- markdownlint-enable no-inline-html -->

_Update 2020-11-26_: I've updated this post to reflect the new [OpenContainer repository location and PHP 7.4 syntax](https://github.com/modethirteen/OpenContainer).

I created [OpenContainer](https://github.com/modethirteen/OpenContainer) to assist in the refactoring of legacy PHP code at [MindTouch](https://mindtouch.com). The goal was to shore up the stability of the codebase as we transitioned from six-month waterfall-driven software delivery cycles to continuous delivery twice per week. While the majority of critical business logic was implemented and exposed via APIs in a C# codebase with (fairly) okay test coverage, PHP was used to marshall data from these APIs and deliver a product experience. There were two major challenges to achieving reasonable stability and reliability for the PHP codebase:

- The codebase lacked any automated unit or integration testing
- Nearly all code relied on static functions or global state, with no formalization in code of the relationship between dependencies

The latter situation, in particular, got under my skin. I've never particularly been a fan of dynamic or loosely typed languages (it bothers me that we are so quick to throw decades worth of type system research out the window for so-called flexibility). Just about every global variable that could be mutated from anywhere across the codebase was mutated, usually as side effects of seemingly unrelated routines. If code paths were going to be tested, I first needed to understand what these paths were and to do that I had to define what was needed (depended upon) for any given scenario.

```php
class XyzzyService {

    public static function newXyzzy() {
        global $wgXyzzySettings;
        return new Xyzzy($wgXyzzySettings);
    }
}

class Foo {

    public static function getXyzzy() {
        global $wgSomeUnrelatedApplicationState, $wgXyzzySettings;
        $wgSomeUnrelatedApplicationState = 'plugh';
        $wgXyzzySettings = array(
            'baz' => 'qux'
        );
        return XyzzyService::getXyzzy();
    }
}
```

Oh yea, there is absolutely nothing wrong with that scenario at all. `XyzzyFactory::newXyzzy` relies on a global variable that is set out-of-band in the function that calls it. `Foo::getXyzzy` cannot set different factory settings for testing and mutates a seemingly unrelated application state variable and ruins some other downstream component's day.

I'm a tremendous fan of dependency injection as a concept (particularly by object constructor). With dependency injection properly leveraged, not only do I understand what the bare minimum requirements are for a software component to work, but the dependencies themselves can be provided as interfaces, with their actual implementations provided by particular use cases.

```php
class XyzzyService implements IXyzzyService {

    private IXyzzySettings $settings;

    public function __constructor(IContainer $container) {
        $settings = $container->IXyzzySettings;
    }

    public function newXyzzy() : IXyzzy {
        return new Xyzzy($this->settings);
    }
}

class Foo {

    private IXyzzySettings $settings;
    private IXyzzyService $service;

    public function __constructor(IContainer $container) {
        $settings = $container->IXyzzySettings;
        $settings->set('baz', 'qux');
        $service = $container->IXyzzyService;
    }

    public function getXyzzy() : IXyzzy {
        return $this->service->newXyzzy();
    }
}
```

Here I have achieved two benefits. `Foo` and `XyzzyService` both clearly define their outside dependencies by fetching them from a shared container. No untraceable global variables are overwritten, and I can step through this code in a debugger. I'd likely have a problem with the `Foo::__construct` method changing the internal state of the container's `IXyzzySettings` instance, but the point is: at least I can track that down and identify when components may be altering state in a way that creates unintended consequences. Furthermore, now that the dependencies for `Foo` and `XyzzyService` are provided as interfaces, I can implement whatever state I want for `IXyzzySettings`. When I test `Foo`, I can even substitute a mock or dummy object for `IXyzzyService`, which leads me into mocking this container - or more specifically, _automocking_.

Automocking is what greatly increased my ability to quickly write tests to cover the behavior that I was converting from static and global implementations. The process went something like this: manually test the desired "hot" paths (based on the original product spec, if it existed), update the code, manually test again, lock down the behavior with a unit test, rinse, and repeat. Automocking removed the need to individually create mocks for all the possible dependencies that could exist in the container. It can be very tedious to identify all injected dependencies in an object and set them up as mocks in the event that they _may_ be needed for a particular test. Failure to do so would often lead to null reference exceptions in the object's constructor when just trying to initialize it for testing.

```php
class Qux {

    public function __constructor(IContainer $container) {
        $bar = $container->IBar
        $value = $container->IFoo->getValue();
    }
}
```

For this scenario, I only need to test `Qux` with different implementations of `IBar`, but this object can't initialize because `IFoo` is not mocked and `getValue` is called on a null reference. That's pretty annoying - I have to mock a dependency I don't care about. If only someone _else_ could do it! 😂

The following sections assume that you have checked out [OpenContainer](https://github.com/modethirteen/OpenContainer) and have familiarized yourself with the library. In short, the key piece we will leverage here is the `@property` PHPDoc value that we use with OpenContainer to create type hint friendly container dependencies (if you have a PHP IDE with intellisense such as [JetBrains PHPStorm](https://www.jetbrains.com/phpstorm)).

```php
use modethirteen\OpenContainer\IContainer;

/**
 * in order for us to automock these properties
 * fully-qualified class names must be included
 *
 * @property \My\Application\IFoo $IFoo
 * @property \My\Application\IBar $IBar
 * @property \My\Application\IQux $IQux
 */
interface IApplicationContainer extends IContainer {
}
```

This container interface contains type hints for all the possible dependencies that could be registered in this container. One drawback to this approach is that it takes some diligence to remember to add a `@property` to the interface every time a new dependency is registered in the container. Our new `MockContainer` will implement this interface and provide some behavior to automatically generate [PHPUnit](https://github.com/sebastianbergmann/phpunit) mocks for these dependencies.

```php
use phpDocumentor\Reflection\DocBlockFactory;
use PHPUnit\Framework\MockObject\Matcher\AnyInvokedCount;
use PHPUnit\Framework\MockObject\MockBuilder;
use PHPUnit\Framework\MockObject\MockObject;
use PHPUnit\Framework\TestCase;
use ReflectionClass;

class MockContainer implements IApplicationContainer {

    /**
     * @var object[]
     */
    private $instances = [];

    /**
     * @var MockObject[]
     * @structure [type] => MockObject
     */
    private $mocks = [];

    /**
     * @var object[]
     * @structure [type] => object
     */
    private $proxyInstances = [];

    /**
     * @var TestCase
     */
    private $testCase;

    /**
     * @var string[]
     * @structure [type] => 'class'
     */
    private $typesToMock = [];

    /**
     * @param TestCase $testCase
     */
    public function __construct(TestCase $testCase) {
        $this->testCase = $testCase;

        // we'll lazily build mocks from interface PHPdoc @property values
        $factory = DocBlockFactory::createInstance();
        $docBlock = $factory
            ->create(
               (new ReflectionClass(IApplicationContainer::class))
                   ->getDocComment()
            );
        if($docBlock === null) {
            return;
        }
        foreach($docBlock->getTagsByName('property') as $tag) {
            [$class, $type] = explode(' ', strval($tag));
            $type = ltrim($type, '$');
            $this->typesToMock[$type] = $class;
        }
    }

    /**
     * Always prefer registered instances to mocks
     *
     * @param string $id
     * @return object
     */
    public function __get(string $id) : object {
        $instance = $this->instances[$id];
        if($instance !== null) {
            return $instance;
        }
        $mock = $this->getMock($id);
        if($mock !== null) {
            return $mock;
        }
        throw new CannotLoadInvalidMockInstanceException($id);
    }

    public function isRegistered(string $id) : bool {
        return true;
    }

    /**
     * Get a mock out of the container to setup expectations
     *
     * @param string $id
     * @return MockObject|null
     */
    public function getMock(string $id) : ?MockObject {
        if(isset($this->mocks[$id])) {
            return $this->mocks[$id];
        }
        $mock = null;
        if(isset($this->typesToMock[$id])) {
            $class = $this->typesToMock[$id];
            $builder = new MockBuilder($this->testCase, $class);
            $mock = $builder
                ->setMethods(get_class_methods($class))
                ->disableOriginalConstructor()
                ->getMock();
            $this->mocks[$id] = $mock;
        }
        return $mock;
    }

    /**
     * Register an instance that provides concrete properties and functions
     * that can be overridden with mocked properties and functions
     *
     * @param string $id
     * @param object $instance
     */
    public function proxyMock(string $id, object $instance) : void {
        if(isset($this->proxyInstances[$id])) {
            $this->proxyInstances[$id] = $instance;
            return;
        }
        $this->proxyInstances[$id] = $instance;
        $mock = $this->getMock($id);

        // proxy each mock method to instance method
        foreach(get_class_methods($instance) as $method) {
            if($method === '__clone' || $method === '__construct') {
                continue;
            }
            $mock->expects(new AnyInvokedCount())
                ->method($method)
                ->willReturnCallback(function() use ($id, $method) {
                    return call_user_func_array([
                        $this->proxyInstances[$id],
                        $method
                    ], func_get_args());
                });
        }
    }

    /**
     * Register a mock object
     *
     * @param string $id
     * @param MockObject $mock
     */
    public function registerMock(string $id, MockObject $mock) : void {
        $this->mocks[$id] = $mock;
    }

    /**
     * Remove any registered instance or mock
     *
     * @param string $id
     */
    function flushInstance(string $id): void {
        unset($this->instances[$id]);
        unset($this->mocks[$id]);
        unset($this->proxyInstances[$id]);
    }

    /**
     * Register a callback that builds an instance
     *
     * @param string $id
     * @param Closure $builder
     */
    function registerBuilder(string $id, Closure $builder): void {
        throw new NotImplementedException();
    }

    /**
     * Register an instance
     *
     * @param string $id
     * @param object $instance
     */
    function registerInstance(string $id, object $instance): void {
        $this->instances[$id] = $instance;
    }

    /**
     * Register a class type
     *
     * @param string $id
     * @param string $class
     */
    function registerType(string $id, string $class): void {
        throw new NotImplementedException();
    }
}
```

Our `MockContainer` uses [phpDocumentor](https://github.com/phpDocumentor/phpDocumentor) and PHPUnit to parse our container interface `@property` values and auto-generate mocks. In our tests, we can now `MockContainer::getMock` any dependency we need to set expectations on. We can use `MockContainer::proxyMock` to provide a concrete instance, with optional mocked properties and functions - a sort of hybrid approach that is influenced by JavaScript testing practices like [spies](https://jasmine.github.io/2.0/introduction#section-Spies).

```php
use PHPUnit\Framework\TestCase;

class FooTest extends TestCase {

    private IApplicationContainer $container;

    public function setUp() : void {
        $this->container = new MockContainer($this);
        $this->container->getMock('IBar')
            ->expects(static::any())
            ->method('getSomething')
            ->will(static::returnValue('123'));
        $this->container->proxyMock('IQux', new Qux());
    }

    /**
     * @test
     */
    public function test() : void {

        // this test has a very specific value it needs IQux to return
        $this->container->getMock('IQux')
            ->expects(static::once())
            ->method('getSomethingElse')
            ->will(static::returnValue('456'));
    }
}
```

Applying these patterns has transformed a codebase that was responsible for a production bug nearly every week, to arguably one of the most covered with tests and most reliable in the entire platform. That level of confidence has a huge benefit on developer morale, especially on those who may be new to the codebase and are concerned with introducing bugs in an unfamiliar environment. Reliable dependency management and testing won't ever entirely eliminate bugs (your unique production situations and data will always see to that), but it can get you very close!
