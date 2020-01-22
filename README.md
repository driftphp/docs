# Architecture

This framework is an exercise of mixing the best of two excellent worlds if we
talk about PHP. On one hand we have Symfony, a collection of PHP components with
a large usage around the PHP ecosystem. On top of them, the Symfony framework,
with a well known project architecture. On the other hand, we find ReactPHP, a
PHP library on top of Promises and an EventLoop, that provides a really easy
non-blocking way of managing Requests using a single PHP thread.

DriftPHP framework follow the Symfony structure, the same Request/Response 
paradigm and the same Kernel bases, but turning all the process non-blocking by
using ReactPHP promises and event loop.

If you're used to working with Symfony skeleton, DriftPHP will seem a bit
familiar then. You will recognise the kernel (a different one, but in fact,
almost the same), the way Controllers and Commands are defined, called and used,
and the way we define services using Dependency Injection (yes, you can still
use autowiring here...).

Before start digging into DriftPHP, is important to understand a little bit what
concepts are you going to use, or even discover, in this package. Each topic is
strongly related to the development of any application on top of DriftPHP, so
make sure you understand each one of them. In the future, all these topics will
be packed into a new and important chapter of the documentation is being built
at this moment.

## Symfony Kernel

Have you ever thought what the Symfony Kernel is really about? In fact, we could
say that is one of the most important elements of the whole Symfony ecosystem,
and we would be wrong if we say that is not the most core one. But what is it
really about? Let's reduce its complexity into 1 simple bullet.

- Given a Request, give me a Response

That's it. Having a Request object, properly hydrated from the magic PHP
variables (`$_GET`, `$_POST`, `$_SERVER` ...), the kernel has one single (and
important) job. Guess everything is needed to return a Response. In order to
make that job, the Kernel uses an event dispatcher to dispatch some events. All
these events are properly documented in the Symfony documentation. And because
of these events and some subscribers included in the `FrameworkBundle` package,
we can match a route, guess the proper Controller instance and call the
Controller method, depending on our configuration.

Once we are in the controller, and if we don't have injected the Container
under any circumstance, then we can say that the Framework job is properly
finished. This is what we can call our domain  

**Welcome to your domain**

So on one side of the framework (probably in your Symfony `/public/index.php`
file) you will find a line where the Kernel handles a Request and expects a
Response.

```php
$response = $kernel->handle($request);
```

Each time you see something like that, take in account that your server will be
blocked from the function call, until the result is returned. This is what we
can define as a blocking operation. If the whole operations lasts 10 seconds,
during 10 seconds nothing will be doable in this PHP thread (yes. 10 seconds.
A slow Elasticsearch operation or MYSql operation could last something like
that, at least, could be possible, right?)

We will change that.
Keep reading.

## HTTP Server

Now remember one of your Symfony projects. Remember about the Apache or Nginx
configuration, then remember about the PHP-FPM, about how to link all together,
about deploying it on docker...

Have you ever thought about what happens there? I'm going to show you a small
bullet list about what's going on.

- Your Nginx takes the HTTP Request.
- The Http Request is passed to PHP-FPM and is interpreted by PHP-FPM.
- Interpreted means, basically, that your `public/index.php` file will be
executed.
- One kernel is instanced and a new Symfony Request is created..
- The Symfony Request is handled and a new Response is generated
- The Response is passed through the Nginx and returned to the final user.

Now, let's ask ourselves a simple question. How many requests do we have per day
in our project? Or even more important. How long it takes to generate a kernel, 
build all the needed services (**all**) once and again, and return the result?

Well. The answer is pretty easy. Depends on how much performance do you really
need. If you have a 1000 requests/day blog, then you will be probably okay with
this stack, but what if you have millions per hour? How important in that case
can be a simple millisecond?

Can we solve this?
Yes. We will solve this.
Keep reading.

## ReactPHP

Since some years ago, PHP turned a bit more interesting with the implementation
of Promises from ReactPHP. The language is exactly the same, PHP, but the unique
difference between the regular PHP usage and the ReactPHP is that each time your
project uses the infrastructure layer (Filesystem, Redis, Mysql) everything is
returned eventually.

About the fundamental problem of using ReactPHP Promises in your regular Symfony
application you can find a little bit more information on these links.

- https://medium.com/@apisearch/symfony-and-reactphp-series-82082167f6fb
- https://es.slideshare.net/MarcMorera/when-symfony-met-promises-167235900

And some links about ReactPHP

- https://reactphp.org
- https://github.com/reactphp

In few words. You cannot use Promises in Symfony, basically because the kernel
is blocking. Having a Request, wait for the Response. Nothing can be `eventual`
here, so even if you use Promises, at the end, you will have to wait for the
response because the Controller **have to** return a Response instance.

Can we solve this?
Again. Yes.
Keep reading.

## The Framework

So, having 3 small problems in front of us, and with the big promise to solve
this situation, here you have the two main packages that this framework offers
to the user. All of them can be used separately, but all together work as is
expected.

- [HTTP Kernel](https://github.com/driftphp/http-kernel)

Simple. Overwrites the regular and blocking Symfony Kernel to a new one, 
inspired by the original, but adapted to work on top of Promises. So everything
here will be non-blocking, and everything will happen *eventually*.

Check the documentation about the small changes you need in your project to use
this kernel. Are pretty small, but completely necessary if you want to start
using Promises in your domain.

- In your Controller, now you will have to return a Promise instead of a
Response
- Your event listeners will have to be packed inside a Promise as well.
Remember, all will happen eventually.

First solved problem

- [HTTP Server](https://github.com/driftphp/server)

Forget about PHP-FPM, about Nginx, Apache or any other external servers. Forget
about them because you will not need them anymore. With this new server, you
will be able to connect your application to a socket directly with no
intermediaries.

And guess what.  
Your kernel will be created once, and only once.  
That means that the first requests will last as long as all your current
requests, but the next ones... well, you won't believe the numbers.

- [Skeleton](https://github.com/driftphp/skeleton)

The skeleton is very simple and basic. Instead of having multiple and different
ways of doing a simple thing, we have defined a unique way of doing everything.
First of all, the main app configuration (Kernel, routes, services and 
bootstrap) is included in the main `Drift/` folder.

You can place all your services wherever you want, but we encourage you to use a
simple and useful architecture based on layers.

```yaml
Drift/ - All your Drift configuration
src/
    - Controller/ - Your controllers. No business logic here
    - Console/ - Your console commands. No logic here
    - Domain/ - Your domain. Only your domain
    - Redis/ - Your Redis adapters
    - Mysql/ - Your Mysql adapters
```

# Getting started

Welcome to the documentation of DriftPHP. Yet another framework on top of PHP,
yes, but in DriftPHP we want to make a deal with you. Performance matters
equally to functionality, so you take care about your domain, and we take care
about serving it in an insane way.

> DriftPHP is not stable. This means that you should not put it on production,
> at least before making a really big testing coverage, specially functional
> testing coverage. We will work to make this collection of packages stable as
> fast as we can.

You can start using DriftPHP by just installing our small skeleton project. In
this skeleton you will be able to start working in a traditional project on top
of the Symfony syntax and Request/Response paradigm.

``` bash
composer create-project drift/skeleton -sdev
```

## Start the server

Once you have created a small skeleton of your application on your computer,
let's run the server. This server is meant to be production-ready, so in this
documentation you will find many ways to deploy this in your infrastructure.

``` bash 
php vendor/bin/server run 0.0.0.0:8000
```

You can start the framework in development mode and enable debugging. Of course,
this configuration is meant to be used only on development.

``` bash 
php vendor/bin/server run 0.0.0.0:8000 --dev --debug
```

And you can even start the server in watcher mode by using the watch command 
instead of the run one. As soon as you make changes on your code, the server
will be reloaded, so you don't have to restart it again once and again.

``` bash 
php vendor/bin/server watch 0.0.0.0:8000 --dev --debug
```

## Test the endpoints

> You can test a couple of endpoints already built-in in this skeleton. Make 
> sure to delete the endpoint before working on a real life project.

Once you have the server connected to a websocket, it's time to check some
endpoints and see if this is already working.

> Make sure your server is properly running

You can test a simple test endpoint...

```bash
curl "127.0.0.1:8000"
```

... or you can test how static content server works by opening in your browser
a distributed image. You have to open in your browser
`http://127.0.0.1:8000/public/driftphp.png` and you should see our amazing logo.

## Events

DriftPHP uses the same kernel events that Symfony does. You can check a bit more
information about them in 
[Built-in Symfony Events](https://symfony.com/doc/current/reference/events.html).
DriftPHP introduces a new kernel event called `kernel.preload`, and dispatched
once the Kernel is booted. This event is dispatched only once and must be used
only for preloading content before the first request is handled.

```php
use Drift\HttpKernel\Event\PreloadEvent;

/**
 * @param PreloadEvent $event
 */
public function onKernelPreload(PreloadEvent $event)
{
    // ...
}
```

On the other hand, the event `kernel.terminate` is never triggered, as the
kernel is never terminated.

Here you have the list of used events in DriftPHP kernel.

- kernel.preload
- kernel.request
- kernel.controller
- kernel.controller_arguments
- kernel.view
- kernel.response
- kernel.finish_request
- kernel.exception

## Built-in services

In DriftPHP you can find some already built-in services, ready to be injected
and used in your services. They are all mostly related to the EventLoop
(remember that you can work only with one running EventLoop, and this one is
already created for the server itself).

### EventLoop

You can use the EventLoop in your services by injecting it, even using
autowiring.

```php
public function __construct(LoopInterface $loop)
{
    $this->loop = $loop;
}
```

You can access to the loop in the container by using this service name

- drift.event_loop

### Filesystem

DriftPHP have created for you a Filesystem instance. Use this object to make all
disk requests, like reading from a file or writing a new one. Do **not** use the
regular PHP methods or your server will experiment a serious lack of performance
due to blocking operations.

You can read a bit more about this package at
[ReactPHP Filesystem](https://github.com/reactphp/filesystem)

```php
use React\Filesystem\Filesystem;

public function __construct(Filesystem $filesystem)
{
    $this->filesystem = $filesystem;
}
```

You can access to the loop in the container by using this service name

- drift.filesystem

## Creating a Controller

This is the easiest part of all. You can create controllers the same way you've
done until now, but the **only** difference is that, hereinafter, you won't
return Response instances, but Promises that, once resolved, will return a
Response object.

Let's take a look at `GetValueController` code of our demo. This is the response
code for the controller called method. This Redis client is asynchronous, so
each time you call any method, for example `->get($key)` you won't get a value,
but a promise. 

```php
namespace App\Controller;

use App\Redis\RedisWrapper;
use React\Promise\PromiseInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;

/**
 * Class GetValueController
 */
class GetValueController
{
    /**
     * @var RedisWrapper
     *
     * Redis Wrapper
     */
    private $redisWrapper;

    /**
     * PutValueController constructor.
     *
     * @param RedisWrapper $redisWrapper
     */
    public function __construct(RedisWrapper $redisWrapper)
    {
        $this->redisWrapper = $redisWrapper;
    }

    /**
     * Invoke
     *
     * @param Request $request
     *
     * @return PromiseInterface
     */
    public function __invoke(Request $request)
    {
        $key = $request
            ->attributes
            ->get('key');

        return $this
            ->redisWrapper
            ->getClient()
            ->get($key)
            ->then(function($value) use ($key) {
                return new JsonResponse(
                    [
                        'key' => $key,
                        'value' => is_string($value)
                            ? $value
                            : null,
                    ],
                    200
                );
            });
    }
}
```

When the promise is resolved, the `then` method is called with the given result
as a unique parameter. Once we have this value, we return a `JsonResponse`
instance inside the callback.

And because a `->then()` method returns a new Promise, and we're returning
directly this value, the kernel receives a Promise and not a value. And that's
it. The server should take care of the rest

## Check the demo

If you want a bit more experience with DriftPHP you might have a look at our
distributed demo. We will update this demo in order to have as many built-in
functionalities and integrations as we can, so make sure you keep up-to-date
with new updates.

- [DriftPHP Demo](https://github.com/driftphp/demo)

Make sure you give this demo a Github star. That will help us a lot.

# The HTTP Kernel

[![CircleCI](https://circleci.com/gh/driftphp/http-kernel.svg?style=svg)](https://circleci.com/gh/driftphp/http-kernel)

This package provides async features to the Symfony (+4.3) Kernel. This
implementation uses [ReactPHP Promise](https://github.com/reactphp/promise) 
library and paradigm for this purposes.

## Installation

You can install the package with composer. This is a PHP Library, so installing
this repository will not change your original project behavior.

```yml
{
  "require": {
    "drift/http-kernel": "dev-master"
  }
}
```

You can get the [source code](https://github.com/driftphp/http-kernel) from 
Github.

Once you have the package under your vendor folder, now it's time to turn you
application asynchronous-friendly by changing your kernel implementation, from
the Symfony regular HTTP Kernel class, to the new Async one.

```php
use Drift\HttpKernel\AsyncKernel;

class Kernel extends AsyncKernel
{
    use MicroKernelTrait;
```

> With this change, nothing should happen. This async kernel maintains all back
> compatibility, so should work inside any synchronous (regular) Symfony
> project.

## Controllers

Your controller will be able to return a Promise now. It is mandatory to do
that? Non-blocking operations are always optional, so if you build your domain
blocking, this is going to work as well.

```php
/**
 * Class Controller.
 */
class Controller
{
    /**
     * Return value.
     *
     * @return Response
     */
    public function getValue(): Response
    {
        return new Response('X');
    }

    /**
     * Return fulfilled promise.
     *
     * @return PromiseInterface
     */
    public function getPromise(): PromiseInterface
    {
        return new FulfilledPromise(new Response('Y'));
    }
}
```

Both controller actions are correct.

## Event Dispatcher

Going asynchronous has some intrinsic effects, and one of these effects is that
event dispatcher has to work a little bit different. If you base all your domain
on top of Promises, your event listeners must be a little bit different. The
events dispatched are exactly the same, but the listeners attached to them must
change a little bit the implementation, depending on the expected behavior.

An event listener can return a Promise. Everything inside this promise will be
executed once the Promise is executed, and everything outside the promise will
be executed at the beginning of all listeners, just before the first one is
fulfilled.

```php
/**
 * Handle get Response.
 *
 * @param ResponseEvent $event
 *
 * @return PromiseInterface
 */
public function handleGetResponsePromiseA(ResponseEvent $event)
{
    $promise = (new FulfilledPromise())
        ->then(function () use ($event) {
        
            // This line is executed eventually after the previous listener
            // promise is fulfilled
        
            $event->setResponse(new Response('A'));
        });
        
    // This line is executed before the first event listener promise is
    // fulfilled
        
    return $promise;
}
```

# The Server

[![CircleCI](https://circleci.com/gh/driftphp/server.svg?style=svg)](https://circleci.com/gh/driftphp/server)

This package provides an async server for DriftPHP framework based on ReactPHP
packages and Promise implementation. The server is distributed with all the
Symfony based kernel adapters, and can be easily extended for new Kernel
modifications.

## Installation

In order to use this server, you only have to add the requirement in composer.
Once updated your dependencies, you will find a brand new server bin inside the
`vendor/bin` folder.

```yml
{
  "require": {
    "driftphp/server": "dev-master"
  }
}
```

You can get the [source code](https://github.com/driftphp/server) from 
Github.

## Usage

This is a PHP file. This means that the way of starting this server is by, just,
executing it.

```console
vendor/bin/server run 0.0.0.0:8100
```

You will find that the server starts with a default configuration. You can
configure how the server starts and what adapters use.

- Adapter: The kernel adapter for the server. This server needs a Kernel
  instance in order to start serving Requests. By default, `symfony4`. Can be
  overridden with option `--adapter` and the value must be a valid class
  namespace of an instance of `KernelAdapter`

```console
php vendor/bin/server run 0.0.0.0:8100 --adapter=symfony4
php vendor/bin/server run 0.0.0.0:8100 --adapter=My\Own\Adapter
```

- Environment: Kernel environment. By default `prod`, but turns `dev` if the
  option `--dev` is found, or the defined one if you define it with `--env`
  option.

```console
php vendor/bin/server run 0.0.0.0:8100 --dev
php vendor/bin/server run 0.0.0.0:8100 --env=test
```

- Debug: Kernel will start with this option is enabled. By default false,
  enabled if the option `--debug` is found. Makes sense on development
  environment, but is not exclusive.

```console
php vendor/bin/server run 0.0.0.0:8100 --dev --debug
```

## Watcher

The server uses a watcher for dynamic content changes. This watches will check
for changes on every file you want, and will reload the server properly.

- Go to [PHP Watcher documentation](https://github.com/seregazhuk/php-watcher)

In order to start the server in a watching mode, use the `watch` command instead
of the `run` command. All the documented options under the `run` command will be
valid in this new command.

```console
php vendor/bin/server watch 0.0.0.0:8100
```

## Serving static files

Kernel Adapters have already defined the static folder related to the kernel.
For example, Symfony4 adapter will provide static files from folder `/public`.

You can override the static folder with the command option `--static-folder`.
All files inside this defined folder will be served statically in a non-blocking
way

```console
php vendor/bin/server run 0.0.0.0:8100 --static-folder=public
```

You can disable static folder with the option `--no-static-folder`. This can be
useful when working with the adapter value and want to disable the default
value, for example, for an API.


```console
php vendor/bin/server run 0.0.0.0:8100 --no-static-folder
```

# ReactPHP functions

[![CircleCI](https://circleci.com/gh/driftphp/reactphp-functions.svg?style=svg)](https://circleci.com/gh/driftphp/reactphp-functions)

Set of simple PHP functions turned non-blocking on too of
[ReactPHP](https://reactphp.org/)

## Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/reactphp-functions) from Github.

```bash
composer require drift/react-functions
```

## Quickstart example

You can easily add a sleep in your non-blocking code by using these functions.
In this code you can wait 1 second before continuing with the execution of your
promise, while the loop is actively switching between active Promises.

```php
    // use a unique event loop instance for all parallel operations
    $loop = React\EventLoop\Factory::create();

    $promise = $this
        ->doWhateverNonBlocking($loop)
        ->then(function() use ($loop) {

            // This will be executed on time t=0
            return React\sleep(1, $loop);
        })
        ->then(function() {
            
            // This will be executed on time t=1
        });
```

## Usage

This lightweight library has some small methods with the exact behavior than
their sibling methods in regular and blocking PHP.

```php
use Drift\React;

React\sleep(...);
```

## EventLoop

Each function is responsible for orchestrating the [EventLoop](https://github.com/reactphp/event-loop#usage)
in order to make it run (block) until your conditions are fulfilled.

```php
$loop = React\EventLoop\Factory::create();
```

> Under DriftPHP framework, you don't have to create a new EventLoop instance,
> but inject the one you have already built, named `@drift.event_loop`.

## sleep

The `sleep($seconds, LoopInterface $loop)` method can be used to sleep for
$time seconds.

```php
React\sleep(1.5, $loop)
    ->then(function() {
        // Do whatever you need after 1.5 seconds
    });
```

It is important to understand all the possible sleep implementations you can use
under this reactive programming environment.

- `\sleep($time)` - Block the PHP thread n seconds, and after this time is
elapsed, continue from the same point
- `\Clue\React\Block\sleep($time, $loop` - Don't block the PHP thread, but let
the loop continue doing cycles. Block the program execution after n seconds, and
after this time is elapsed, continue from the same point. This is a blocking
feature.
- `\Drift\React\sleep($time, $loop)` - Don't block neither the PHP thread nor
the program execution. This method returns a Promise that will be resolved after
n seconds. This is a non-blocking feature.

## usleep

The `sleep($seconds, LoopInterface $loop)` method can be used to sleep for
$time microseconds.

```php
React\usleep(3000, $loop)
    ->then(function() {
        // Do whatever you need after 3000 microseconds
    });
```

The same rationale than the [`React\sleep`](#sleep) method. This is a
non-blocking action.

## mime_content_type

The `mime_content_type("/tmp/file.png", LoopInterface $loop)` method can be used
to guess the mime content type of a file. If failure, then rejects with a
RuntimeException.

```php
React\mime_content_type("/tmp/file.png", $loop)
    ->then(function(string $type) {
        // Do whatever you need with the found mime type
    });
```

This is a non-blocking action and equals the regular PHP function
[mime_content_type](https://www.php.net/manual/en/function.mime-content-type.php).

# Adapters

In order to be able to use DriftPHP and your whole domain in a non-blocking way,
all your infrastructure operation must be done by using ReactPHP libraries. This
framework, and taking the advantage of Symfony bundle architecture, offers you a
set of components to make you configure these clients in a more easy and
structured way.

## Redis adapter

[![CircleCI](https://circleci.com/gh/driftphp/redis-bundle.svg?style=svg)](https://circleci.com/gh/driftphp/redis-bundle)

This is a simple adapter for Redis on top of ReactPHP and DriftPHP. Following
the same structure that is followed in the Symfony ecosystem, you can use this
package as a Bundle, only usable under DriftPHP Framework.

### Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/redis-bundle) from Github.

```bash
composer require drift/redis-bundle
```

### Configure

This package will allow you to configure all your Redis async clients, taking
care of duplicity and the loop integration. Once your package is required by
composer, add the bundle in the kernel and change your `services.yaml`
configuration file to defined the clients

```yaml
redis:
    clients:
        users:
            host: "127.0.0.1"
            port: 6379
            database: "/users"
            password: "secret"
            protocol: "rediss://"
            idle: 0.5
            timeout: 10.0
        orders:
            host: "127.0.0.1"
            database: "/orders"
```

Only host is required. All the other values are optional.

### Usage

Once you have your clients created, you can inject them in your services by
using the name of the client in your dependency injection arguments array

```yaml
a_service:
    class: My\Service
    arguments:
        - "@redis.users_client"
        - "@redis.orders_client"
```

You can use Autowiring as well in the bundle, by using the name of the client
and using it as a named parameter

```php
use Clue\React\Redis\Client;

public function __construct(
    Client $usersClient,
    Client $ordersClient
)
```

## Mysql adapter

[![CircleCI](https://circleci.com/gh/driftphp/mysql-bundle.svg?style=svg)](https://circleci.com/gh/driftphp/mysql-bundle)

This is a simple adapter for Mysql on top of ReactPHP and DriftPHP. Following
the same structure that is followed in the Symfony ecosystem, you can use this
package as a Bundle, only usable under DriftPHP Framework.

### Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/mysql-bundle) from Github.

```bash
composer require drift/mysql-bundle
```

### Configure

This package will allow you to configure all your Mysql async clients, taking
care of duplicity and the loop integration. Once your package is required by
composer, add the bundle in the kernel and change your `services.yaml`
configuration file to defined the connections

```yaml
mysql:
    connections:
        users:
            host: "127.0.0.1"
            port: 3306
            user: "root"
            password: "root"
            database: "users"
        orders:
            host: "127.0.0.1"
            user: "root"
            password: "root"
            database: "orders"
```

All parameters are required, but the port. This one is `3306` by default.

### Usage

Once you have your connections created, you can inject them in your services by
using the name of the connection in your dependency injection arguments array

```yaml
a_service:
    class: My\Service
    arguments:
        - "@mysql.users_connection"
        - "@mysql.orders_connection"
```

You can use Autowiring as well in the bundle, by using the name of the 
connection and using it as a named parameter

```php
use React\MySQL\ConnectionInterface;
use React\MySQL\Io\LazyConnection;

public function __construct(
    ConnectionInterface $usersConnection,
    LazyConnection $ordersConnection
)
```

## Twig adapter

[![CircleCI](https://circleci.com/gh/driftphp/twig-bundle.svg?style=svg)](https://circleci.com/gh/driftphp/twig-bundle)

This is a simple adapter for Twig on top of ReactPHP and DriftPHP. Following
the same structure that is followed in the Symfony ecosystem, you can use this
package as a Bundle, only usable under DriftPHP Framework.

This package will work for you as a simple bridge between your DriftPHP and Twig
as a templating engine. Of course, the behavior will be exactly the same than
working on top of other frameworks, like Symfony, but some small changes have
been introduced here in order to be non-blocking.

### Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/twig-bundle) from Github.

```bash
composer require drift/twig-bundle
```

### Usage

You have two different ways of using Twig in DriftPHP. Both ways are pretty
similar, but could be discussed about the grade of coupling that we want between
our layers.

The first one is the regular one. We can inject Twig as a dependency, and having
the path of our template, we can easily render our view. You can check here a
small example about this strategy. As you can see, your controller turns
dependent on a view reference and on the complete Twig engine, which can make
sense somehow.

```php
/**
 * Class ViewValuesController
 */
class ViewValuesController
{
    private $twig;

    /**
     * PutValueController constructor.
     *
     * @param Environment $twig
     */
    public function __construct(Environment $twig)
    {
        $this->twig = $twig;
    }

    /**
     * Invoke
     *
     * @param Request $request
     *
     * @return Response
     */
    public function __invoke(Request $request)
    {
        $template = $this
            ->twig
            ->load('redis/view_values.twig');
        $values = ['drift', 'php'];

        return new Response(
            $template->render([
                'values' => $values
            ]),
            200
        );
    }
}
```

On the other hand, you can make your controller **only** a Twig coupled part by
implementing a small interface, which will force you defining this template path
without having to inject the Environment. In that case, you can return an array
and the bundle will render properly the template.

```php
use Drift\Twig\Controller\RenderableController;

/**
 * Class ViewValueController
 */
class ViewValueController implements RenderableController
{
    /**
     * Invoke
     *
     * @param Request $request
     *
     * @return array
     */
    public function __invoke(Request $request) : array
    {
        return [
            'key' => 'value'
        ];
    }

    /**
     * Get render template
     *
     * @return string
     */
    public static function getTemplatePath(): string
    {
        return 'redis/view_value.twig';
    }
}
```

If you follow that strategy, then you will be able to return promises instead of
simple values. In that case, the bundle will wait until all promises are
properly fulfilled and will render the template.

```php
use Drift\Twig\Controller\RenderableController;

/**
 * Class ViewValueController
 */
class ViewValueController implements RenderableController
{
    /**
     * Invoke
     *
     * @param Request $request
     *
     * @return array
     */
    public function __invoke(Request $request) : array
    {
        return [
            'key' => (new FulfilledPromise())
                ->then(function() {
                    return 'value';
                })
        ];
    }

    /**
     * Get render template
     *
     * @return string
     */
    public static function getTemplatePath(): string
    {
        return 'redis/view_value.twig';
    }
}
```
=======
# Architecture

This framework is an exercise of mixing the best of two excellent worlds if we
talk about PHP. On one hand we have Symfony, a collection of PHP components with
a large usage around the PHP ecosystem. On top of them, the Symfony framework,
with a well known project architecture. On the other hand, we find ReactPHP, a
PHP library on top of Promises and an EventLoop, that provides a really easy
non-blocking way of managing Requests using a single PHP thread.

DriftPHP framework follow the Symfony structure, the same Request/Response 
paradigm and the same Kernel bases, but turning all the process non-blocking by
using ReactPHP promises and event loop.

If you're used to working with Symfony skeleton, DriftPHP will seem a bit
familiar then. You will recognise the kernel (a different one, but in fact,
almost the same), the way Controllers and Commands are defined, called and used,
and the way we define services using Dependency Injection (yes, you can still
use autowiring here...).

Before start digging into DriftPHP, is important to understand a little bit what
concepts are you going to use, or even discover, in this package. Each topic is
strongly related to the development of any application on top of DriftPHP, so
make sure you understand each one of them. In the future, all these topics will
be packed into a new and important chapter of the documentation is being built
at this moment.

## Symfony Kernel

Have you ever thought what the Symfony Kernel is really about? In fact, we could
say that is one of the most important elements of the whole Symfony ecosystem,
and we would be wrong if we say that is not the most core one. But what is it
really about? Let's reduce its complexity into 1 simple bullet.

- Given a Request, give me a Response

That's it. Having a Request object, properly hydrated from the magic PHP
variables (`$_GET`, `$_POST`, `$_SERVER` ...), the kernel has one single (and
important) job. Guess everything is needed to return a Response. In order to
make that job, the Kernel uses an event dispatcher to dispatch some events. All
these events are properly documented in the Symfony documentation. And because
of these events and some subscribers included in the `FrameworkBundle` package,
we can match a route, guess the proper Controller instance and call the
Controller method, depending on our configuration.

Once we are in the controller, and if we don't have injected the Container
under any circumstance, then we can say that the Framework job is properly
finished. This is what we can call our domain  

**Welcome to your domain**

So on one side of the framework (probably in your Symfony `/public/index.php`
file) you will find a line where the Kernel handles a Request and expects a
Response.

```php
$response = $kernel->handle($request);
```

Each time you see something like that, take in account that your server will be
blocked from the function call, until the result is returned. This is what we
can define as a blocking operation. If the whole operations lasts 10 seconds,
during 10 seconds nothing will be doable in this PHP thread (yes. 10 seconds.
A slow Elasticsearch operation or MYSql operation could last something like
that, at least, could be possible, right?)

We will change that.
Keep reading.

## HTTP Server

Now remember one of your Symfony projects. Remember about the Apache or Nginx
configuration, then remember about the PHP-FPM, about how to link all together,
about deploying it on docker...

Have you ever thought about what happens there? I'm going to show you a small
bullet list about what's going on.

- Your Nginx takes the HTTP Request.
- The Http Request is passed to PHP-FPM and is interpreted by PHP-FPM.
- Interpreted means, basically, that your `public/index.php` file will be
executed.
- One kernel is instanced and a new Symfony Request is created..
- The Symfony Request is handled and a new Response is generated
- The Response is passed through the Nginx and returned to the final user.

Now, let's ask ourselves a simple question. How many requests do we have per day
in our project? Or even more important. How long it takes to generate a kernel, 
build all the needed services (**all**) once and again, and return the result?

Well. The answer is pretty easy. Depends on how much performance do you really
need. If you have a 1000 requests/day blog, then you will be probably okay with
this stack, but what if you have millions per hour? How important in that case
can be a simple millisecond?

Can we solve this?
Yes. We will solve this.
Keep reading.

## ReactPHP

Since some years ago, PHP turned a bit more interesting with the implementation
of Promises from ReactPHP. The language is exactly the same, PHP, but the unique
difference between the regular PHP usage and the ReactPHP is that each time your
project uses the infrastructure layer (Filesystem, Redis, Mysql) everything is
returned eventually.

About the fundamental problem of using ReactPHP Promises in your regular Symfony
application you can find a little bit more information on these links.

- https://medium.com/@apisearch/symfony-and-reactphp-series-82082167f6fb
- https://es.slideshare.net/MarcMorera/when-symfony-met-promises-167235900

And some links about ReactPHP

- https://reactphp.org
- https://github.com/reactphp

In few words. You cannot use Promises in Symfony, basically because the kernel
is blocking. Having a Request, wait for the Response. Nothing can be `eventual`
here, so even if you use Promises, at the end, you will have to wait for the
response because the Controller **have to** return a Response instance.

Can we solve this?
Again. Yes.
Keep reading.

## The Framework

So, having 3 small problems in front of us, and with the big promise to solve
this situation, here you have the two main packages that this framework offers
to the user. All of them can be used separately, but all together work as is
expected.

- [HTTP Kernel](https://github.com/driftphp/http-kernel)

Simple. Overwrites the regular and blocking Symfony Kernel to a new one, 
inspired by the original, but adapted to work on top of Promises. So everything
here will be non-blocking, and everything will happen *eventually*.

Check the documentation about the small changes you need in your project to use
this kernel. Are pretty small, but completely necessary if you want to start
using Promises in your domain.

- In your Controller, now you will have to return a Promise instead of a
Response
- Your event listeners will have to be packed inside a Promise as well.
Remember, all will happen eventually.

First solved problem

- [HTTP Server](https://github.com/driftphp/server)

Forget about PHP-FPM, about Nginx, Apache or any other external servers. Forget
about them because you will not need them anymore. With this new server, you
will be able to connect your application to a socket directly with no
intermediaries.

And guess what.  
Your kernel will be created once, and only once.  
That means that the first requests will last as long as all your current
requests, but the next ones... well, you won't believe the numbers.

- [Skeleton](https://github.com/driftphp/skeleton)

The skeleton is very simple and basic. Instead of having multiple and different
ways of doing a simple thing, we have defined a unique way of doing everything.
First of all, the main app configuration (Kernel, routes, services and 
bootstrap) is included in the main `Drift/` folder.

You can place all your services wherever you want, but we encourage you to use a
simple and useful architecture based on layers.

```yaml
Drift/ - All your Drift configuration
src/
    - Controller/ - Your controllers. No business logic here
    - Console/ - Your console commands. No logic here
    - Domain/ - Your domain. Only your domain
    - Redis/ - Your Redis adapters
    - Mysql/ - Your Mysql adapters
```

# Getting started

Welcome to the documentation of DriftPHP. Yet another framework on top of PHP,
yes, but in DriftPHP we want to make a deal with you. Performance matters
equally to functionality, so you take care about your domain, and we take care
about serving it in an insane way.

> DriftPHP is not stable. This means that you should not put it on production,
> at least before making a really big testing coverage, specially functional
> testing coverage. We will work to make this collection of packages stable as
> fast as we can.

You can start using DriftPHP by just installing our small skeleton project. In
this skeleton you will be able to start working in a traditional project on top
of the Symfony syntax and Request/Response paradigm.

``` bash
composer create-project drift/skeleton -sdev
```

## Start the server

Once you have created a small skeleton of your application on your computer,
let's run the server. This server is meant to be production-ready, so in this
documentation you will find many ways to deploy this in your infrastructure.

``` bash 
php vendor/bin/server run 0.0.0.0:8000
```

You can start the framework in development mode and enable debugging. Of course,
this configuration is meant to be used only on development.

``` bash 
php vendor/bin/server run 0.0.0.0:8000 --dev --debug
```

And you can even start the server in watcher mode by using the watch command 
instead of the run one. As soon as you make changes on your code, the server
will be reloaded, so you don't have to restart it again once and again.

``` bash 
php vendor/bin/server watch 0.0.0.0:8000 --dev --debug
```

## Test the endpoints

> You can test a couple of endpoints already built-in in this skeleton. Make 
> sure to delete the endpoint before working on a real life project.

Once you have the server connected to a websocket, it's time to check some
endpoints and see if this is already working.

> Make sure your server is properly running

You can test a simple test endpoint...

```bash
curl "127.0.0.1:8000"
```

... or you can test how static content server works by opening in your browser
a distributed image. You have to open in your browser
`http://127.0.0.1:8000/public/driftphp.png` and you should see our amazing logo.

## Events

DriftPHP uses the same kernel events that Symfony does. You can check a bit more
information about them in 
[Built-in Symfony Events](https://symfony.com/doc/current/reference/events.html).
DriftPHP introduces a new kernel event called `kernel.preload`, and dispatched
once the Kernel is booted. This event is dispatched only once and must be used
only for preloading content before the first request is handled.

```php
use Drift\HttpKernel\Event\PreloadEvent;

/**
 * @param PreloadEvent $event
 */
public function onKernelPreload(PreloadEvent $event)
{
    // ...
}
```

On the other hand, the event `kernel.terminate` is never triggered, as the
kernel is never terminated.

Here you have the list of used events in DriftPHP kernel.

- kernel.preload
- kernel.request
- kernel.controller
- kernel.controller_arguments
- kernel.view
- kernel.response
- kernel.finish_request
- kernel.exception

## Built-in services

In DriftPHP you can find some already built-in services, ready to be injected
and used in your services. They are all mostly related to the EventLoop
(remember that you can work only with one running EventLoop, and this one is
already created for the server itself).

### EventLoop

You can use the EventLoop in your services by injecting it, even using
autowiring.

```php
public function __construct(LoopInterface $loop)
{
    $this->>loop = $loop;
}
```

You can access to the loop in the container by using this service name

- drift.event_loop

### Filesystem

DriftPHP have created for you a Filesystem instance. Use this object to make all
disk requests, like reading from a file or writing a new one. Do **not** use the
regular PHP methods or your server will experiment a serious lack of performance
due to blocking operations.

You can read a bit more about this package at
[ReactPHP Filesystem](https://github.com/reactphp/filesystem)

```php
use React\Filesystem\Filesystem;

public function __construct(Filesystem $filesystem)
{
    $this->>filesystem = $filesystem;
}
```

You can access to the loop in the container by using the service name
`drift.filesystem`.

## Creating a Controller

This is the easiest part of all. You can create controllers the same way you've
done until now, but the **only** difference is that, hereinafter, you won't
return Response instances, but Promises that, once resolved, will return a
Response object.

Let's take a look at `GetValueController` code of our demo. This is the response
code for the controller called method. This Redis client is asynchronous, so
each time you call any method, for example `->get($key)` you won't get a value,
but a promise. 

```php
namespace App\Controller;

use App\Redis\RedisWrapper;
use React\Promise\PromiseInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;

/**
 * Class GetValueController
 */
class GetValueController
{
    /**
     * @var RedisWrapper
     *
     * Redis Wrapper
     */
    private $redisWrapper;

    /**
     * PutValueController constructor.
     *
     * @param RedisWrapper $redisWrapper
     */
    public function __construct(RedisWrapper $redisWrapper)
    {
        $this->redisWrapper = $redisWrapper;
    }

    /**
     * Invoke
     *
     * @param Request $request
     *
     * @return PromiseInterface
     */
    public function __invoke(Request $request)
    {
        $key = $request
            ->attributes
            ->get('key');

        return $this
            ->redisWrapper
            ->getClient()
            ->get($key)
            ->then(function($value) use ($key) {
                return new JsonResponse(
                    [
                        'key' => $key,
                        'value' => is_string($value)
                            ? $value
                            : null,
                    ],
                    200
                );
            });
    }
}
```

When the promise is resolved, the `then` method is called with the given result
as a unique parameter. Once we have this value, we return a `JsonResponse`
instance inside the callback.

And because a `->then()` method returns a new Promise, and we're returning
directly this value, the kernel receives a Promise and not a value. And that's
it. The server should take care of the rest

## Check the demo

If you want a bit more experience with DriftPHP you might have a look at our
distributed demo. We will update this demo in order to have as many built-in
functionalities and integrations as we can, so make sure you keep up-to-date
with new updates.

- [DriftPHP Demo](https://github.com/driftphp/demo)

Make sure you give this demo a Github star. That will help us a lot.

# The HTTP Kernel

[![CircleCI](https://circleci.com/gh/driftphp/http-kernel.svg?style=svg)](https://circleci.com/gh/driftphp/http-kernel)

This package provides async features to the Symfony (+4.3) Kernel. This
implementation uses [ReactPHP Promise](https://github.com/reactphp/promise) 
library and paradigm for this purposes.

## Installation

You can install the package with composer. This is a PHP Library, so installing
this repository will not change your original project behavior.

```yml
{
  "require": {
    "drift/http-kernel": "dev-master"
  }
}
```

You can get the [source code](https://github.com/driftphp/http-kernel) from 
Github.

Once you have the package under your vendor folder, now it's time to turn you
application asynchronous-friendly by changing your kernel implementation, from
the Symfony regular HTTP Kernel class, to the new Async one.

```php
use Drift\HttpKernel\AsyncKernel;

class Kernel extends AsyncKernel
{
    use MicroKernelTrait;
```

> With this change, nothing should happen. This async kernel maintains all back
> compatibility, so should work inside any synchronous (regular) Symfony
> project.

## Controllers

Your controller will be able to return a Promise now. It is mandatory to do
that? Non-blocking operations are always optional, so if you build your domain
blocking, this is going to work as well.

```php
/**
 * Class Controller.
 */
class Controller
{
    /**
     * Return value.
     *
     * @return Response
     */
    public function getValue(): Response
    {
        return new Response('X');
    }

    /**
     * Return fulfilled promise.
     *
     * @return PromiseInterface
     */
    public function getPromise(): PromiseInterface
    {
        return new FulfilledPromise(new Response('Y'));
    }
}
```

Both controller actions are correct.

## Event Dispatcher

Going asynchronous has some intrinsic effects, and one of these effects is that
event dispatcher has to work a little bit different. If you base all your domain
on top of Promises, your event listeners must be a little bit different. The
events dispatched are exactly the same, but the listeners attached to them must
change a little bit the implementation, depending on the expected behavior.

An event listener can return a Promise. Everything inside this promise will be
executed once the Promise is executed, and everything outside the promise will
be executed at the beginning of all listeners, just before the first one is
fulfilled.

```php
/**
 * Handle get Response.
 *
 * @param ResponseEvent $event
 *
 * @return PromiseInterface
 */
public function handleGetResponsePromiseA(ResponseEvent $event)
{
    $promise = (new FulfilledPromise())
        ->then(function () use ($event) {
        
            // This line is executed eventually after the previous listener
            // promise is fulfilled
        
            $event->setResponse(new Response('A'));
        });
        
    // This line is executed before the first event listener promise is
    // fulfilled
        
    return $promise;
}
```

# The Server

[![CircleCI](https://circleci.com/gh/driftphp/server.svg?style=svg)](https://circleci.com/gh/driftphp/server)

This package provides an async server for DriftPHP framework based on ReactPHP
packages and Promise implementation. The server is distributed with all the
Symfony based kernel adapters, and can be easily extended for new Kernel
modifications.

## Installation

In order to use this server, you only have to add the requirement in composer.
Once updated your dependencies, you will find a brand new server bin inside the
`vendor/bin` folder.

```yml
{
  "require": {
    "driftphp/server": "dev-master"
  }
}
```

You can get the [source code](https://github.com/driftphp/server) from 
Github.

## Usage

This is a PHP file. This means that the way of starting this server is by, just,
executing it.

```console
vendor/bin/server run 0.0.0.0:8100
```

You will find that the server starts with a default configuration. You can
configure how the server starts and what adapters use.

- Adapter: The kernel adapter for the server. This server needs a Kernel
  instance in order to start serving Requests. By default, `symfony4`. Can be
  overridden with option `--adapter` and the value must be a valid class
  namespace of an instance of `KernelAdapter`

```console
php vendor/bin/server run 0.0.0.0:8100 --adapter=symfony4
php vendor/bin/server run 0.0.0.0:8100 --adapter=My\Own\Adapter
```

- Environment: Kernel environment. By default `prod`, but turns `dev` if the
  option `--dev` is found, or the defined one if you define it with `--env`
  option.

```console
php vendor/bin/server run 0.0.0.0:8100 --dev
php vendor/bin/server run 0.0.0.0:8100 --env=test
```

- Debug: Kernel will start with this option is enabled. By default false,
  enabled if the option `--debug` is found. Makes sense on development
  environment, but is not exclusive.

```console
php vendor/bin/server run 0.0.0.0:8100 --dev --debug
```

## Watcher

The server uses a watcher for dynamic content changes. This watches will check
for changes on every file you want, and will reload the server properly.

- Go to [PHP Watcher documentation](https://github.com/seregazhuk/php-watcher)

In order to start the server in a watching mode, use the `watch` command instead
of the `run` command. All the documented options under the `run` command will be
valid in this new command.

```console
php vendor/bin/server watch 0.0.0.0:8100
```

## Serving static files

Kernel Adapters have already defined the static folder related to the kernel.
For example, Symfony4 adapter will provide static files from folder `/public`.

You can override the static folder with the command option `--static-folder`.
All files inside this defined folder will be served statically in a non-blocking
way

```console
php vendor/bin/server run 0.0.0.0:8100 --static-folder=public
```

You can disable static folder with the option `--no-static-folder`. This can be
useful when working with the adapter value and want to disable the default
value, for example, for an API.


```console
php vendor/bin/server run 0.0.0.0:8100 --no-static-folder
```

# The Command Bus

[![CircleCI](https://circleci.com/gh/driftphp/command-bus-bundle.svg?style=svg)](https://circleci.com/gh/driftphp/command-bus-bundle)

DriftPHP is a performance-oriented framework on top of PHP. This means that we,
as a project, encourage you to take care of the architecture of all our
implementations. Not only on terms of returning times, but on terms of
scalability as well.

One of the DriftPHP components brings you the opportunity to work on top of CQRS
pattern, using Commands and Queries, some command buses and an already built-in
async adapters with different implementations.

## Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/command-bus-bundle) from Github.

```bash
composer require drift/command-bus
```

Once the package is properly loaded, make sure the bundle is loaded for all the
environments. You should add the bundle in `Drift/config/bundles.php`.

```php
return [
    //
    Drift\CommandBus\CommandBusBundle::class => ['all' => true],
    //
];
```

Once the bundle is loaded in the kernel, you'll be able to use 3 different
buses, each one of them with an specific purpose.

- [QueryBus](#query-bus)
- [CommandBus](#command-bus)
- [InlineCommandBus](#inline-command-bus)

## Query Bus

Use this bus for asking queries. Remember that in the CQRS pattern, queries
should never change the state of the persistence layer and should return a value
or a Domain exception.

```php
use Drift\CommandBus\Bus\QueryBus;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

public function __execute(Request $request) {
    /* @var QueryBus $queryBus */
    return $queryBus
        ->ask(new GetSomething())
        ->then(function($something) {
        
            return new Response($something, 200);
        });
}
```

You can use *autowiring* for the QueryBus injection or you can manually inject
the bus by using it's service name `drift.query_bus`

```yaml
My\Service:
    arguments:
      - "@drift.query_bus"
```

You can add handlers to this bus by defining them with the tag `query_handler`.
Remember that you can define them all in bulk by using *autowiring* and setting
this tag just one time.

```yaml
Domain\QueryHandler\:
    resource: "../../src/Domain/QueryHandler"
    tags: ['query_handler']
```

## Command Bus

Use this bus for making changes to the persistence layer. In this case, CQRS
pattern tells that Commands do **NOT** return any value, and if you return a
Domain exception, this one should not be related to the persistence layer.

```php
use Drift\CommandBus\Bus\CommandBus;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

public function __execute(Request $request) {
    /* @var CommandBus $commandBus */
    return $commandBus
        ->execute(new DoSomething())
        ->then(function() {
        
            return new Response('OK', 202);
        });
}
```

You can use *autowiring* for the CommandBus injection or you can manually inject
the bus by using it's service name `drift.command_bus`

```yaml
My\Service:
    arguments:
      - "@drift.command_bus"
```

You can add handlers to this bus by defining them with the tag `command_handler`.
Remember that you can define them all in bulk by using *autowiring* and setting
this tag just one time.

```yaml
Domain\CommandHandler\:
    resource: "../../src/Domain/CommandHandler"
    tags: ['command_handler']
```

This bus can be configured as async. That means that all your commands will be
extracted from the bus at the point you chose and enqueued somewhere. You'll be
able to consume these commands with a consumer in a non-blocking way.

## Inline Command Bus

This bus will contain exactly the same handlers and middlewares than the last
one, but in this case, the async middleware will not be included never. Use this
bus for commands that you'd never like to be removed from the bus and enqueued
for asynchronous consumption.

```php
use Drift\CommandBus\Bus\InlineCommandBus;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

public function __execute(Request $request) {
    /* @var InlineCommandBus $inlineCommandBus */
    return $inlineCommandBus
        ->execute(new DoSomething())
        ->then(function() {
        
            return new Response('OK', 202);
        });
}
```

You can use *autowiring* for the InlineCommandBus injection or you can manually
inject the bus by using it's service name `drift.inline_command_bus`

```yaml
My\Service:
    arguments:
      - "@drift.inline_command_bus"
```

## Middleware

All these buses can have middleware classes defined in your application or
domain layer. Middleware classes are just simple PHP classes with one public
method. In DriftPHP, unlike other bus implementations, the middleware will be
able to be part of your domain, so implementing an external interface will not
be a thing.

- Where is the contract then? you might ask...

Well, is true that a contract should exist between the bus and your domain, and
the configuration should know what method should be called as soon as is
required.

Well. In DriftPHP you just configure the name of the service and the name of the
method to be called, and if the Middleware is not properly configured, then an
Exception will be thrown during the Kernel build stage.

Let's see how you can append middleware services in your buses.

```yaml
command_bus:
    query_bus:
        middlewares:
          - middleware1.service
          - Another\Service

    command_bus:
        middlewares:
          - middleware1.service
          - Another\Service
```

Remember that both the CommandBus and the InlineCommandBus will share all
configuration.

## Asynchronous Bus

By default, the CommandBus asynchronous property is completely disabled. You can
enable it by choosing one of the proposed implementations. All of them are
implemented using the *Producer/Consumer* pattern.

- Redis : In order to use this implementation, you will have to require and
configure `drift/redis-bundle` package as is described at
[Redis Adapter](#redis-adapter)

In this example, we create a redis client named `queues` that will connect to
localhost. Then we define that our command bus will be asynchronous and will
use this given redis client to use it as a queue system, using the `commands`
redis key.

```yaml
redis:
    clients:
        queues:
            host: 127.0.0.1

command_bus:
    command_bus:
        async_adapter:
            redis:
                client: queues
                key: comamnds
```

- AMQP (RabbitMQ) : In order to use this implementation, you will have to require
and configure `drift/amqp-bundle` package as is described at
[AMQP Adapter](#amqp-adapter)

In this example, we create an amqp client named `queues` that will connect to
localhost. Then we define that our command bus will be asynchronous and will
use this given amqp client to use it as a queue system, using the `commands`
amqp queue.

```yaml
amqp:
    connections:
        queues:
            host: 127.0.0.1

command_bus:
    command_bus:
        async_adapter:
            redis:
                connection: queues
                queue: comamnds
```

- InMemory : You can use this adapter only for testing purposes. Commands are
appended in an internal array, and can be consulted during the testing case. You
can retrieve the adapter by getting the service 
`Drift\CommandBus\Async\InMemoryAdapter` defined as public by default.

```yaml
command_bus:
    command_bus:
        async_adapter:
            in_memory:
```

By default, if you only have one strategy defined, this one will be the chosen
one. If you have many of them, the first one will be the used one. You can
change this behavior and specifically select which one you want to use by
setting it in configuration.

```yaml
command_bus:
    command_bus:
        async_adapter:
            adapter: redis
            in_memory:
            redis:
                connection: queues
                queue: comamnds
```

By default, the async adapter will be prepended in the beginning of the bus. You
can change where you want to add this middleware by hand in the middleware list
in configuration

```yaml
command_bus:
    command_bus:
        middlewares:
          - middleware1.service
          - @async
          - Another\Service
```

In this case, the first `middleware1.service` middleware will be executed before
the command is extracted from the bus, and will be executed too once the command
is consumed asynchronously eventually.

> You might have the option of choosing what commands should be executed
> asynchronously and what commands should be executed inline. Well, this chose
> should be done in the application layer (Controllers, Commands), so make sure
> you inject CommandBus or InlineCommandBus depending on that needs.
>

## Command Consumer

If we have used an async adapter for enqueue all (or just some) commands, then
we should have a mechanism where to consume these commands. And good new are
that this bundle already have this consumer available and ready to use.

As consumed commands shouldn't be enqueued again unless something bad happens,
instead of using CommandBus we will use InlineCommandBus (remember, same 
configuration and async disabled).

To start consuming commands, just use console command. In order to take care
about memory consumption, you can make this consumer valid only during *n*
iterations. After these iterations, the consumer will die. Only valid if you
have a supervisor behind or similar, checking the number of instances running.

```bash
php bin/console bus:consume-commands --limit 10
```

By default, no limit.

# ReactPHP functions

[![CircleCI](https://circleci.com/gh/driftphp/reactphp-functions.svg?style=svg)](https://circleci.com/gh/driftphp/reactphp-functions)

Set of simple PHP functions turned non-blocking on too of
[ReactPHP](https://reactphp.org/)

## Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/reactphp-functions) from Github.

```bash
composer require drift/react-functions
```

## Quickstart example

You can easily add a sleep in your non-blocking code by using these functions.
In this code you can wait 1 second before continuing with the execution of your
promise, while the loop is actively switching between active Promises.

```php
    // use a unique event loop instance for all parallel operations
    $loop = React\EventLoop\Factory::create();

    $promise = $this
        ->doWhateverNonBlocking($loop)
        ->then(function() use ($loop) {

            // This will be executed on time t=0
            return React\sleep(1, $loop);
        })
        ->then(function() {
            
            // This will be executed on time t=1
        });
```

## Usage

This lightweight library has some small methods with the exact behavior than
their sibling methods in regular and blocking PHP.

```php
use Drift\React;

React\sleep(...);
```

## EventLoop

Each function is responsible for orchestrating the [EventLoop](https://github.com/reactphp/event-loop#usage)
in order to make it run (block) until your conditions are fulfilled.

```php
$loop = React\EventLoop\Factory::create();
```

> Under DriftPHP framework, you don't have to create a new EventLoop instance,
> but inject the one you have already built, named `@drift.event_loop`.

## sleep

The `sleep($seconds, LoopInterface $loop)` method can be used to sleep for
$time seconds.

```php
React\sleep(1.5, $loop)
    ->then(function() {
        // Do whatever you need after 1.5 seconds
    });
```

It is important to understand all the possible sleep implementations you can use
under this reactive programming environment.

- `\sleep($time)` - Block the PHP thread n seconds, and after this time is
elapsed, continue from the same point
- `\Clue\React\Block\sleep($time, $loop` - Don't block the PHP thread, but let
the loop continue doing cycles. Block the program execution after n seconds, and
after this time is elapsed, continue from the same point. This is a blocking
feature.
- `\Drift\React\sleep($time, $loop)` - Don't block neither the PHP thread nor
the program execution. This method returns a Promise that will be resolved after
n seconds. This is a non-blocking feature.

## usleep

The `sleep($seconds, LoopInterface $loop)` method can be used to sleep for
$time microseconds.

```php
React\usleep(3000, $loop)
    ->then(function() {
        // Do whatever you need after 3000 microseconds
    });
```

The same rationale than the [`React\sleep`](#sleep) method. This is a
non-blocking action.

## mime_content_type

The `mime_content_type("/tmp/file.png", LoopInterface $loop)` method can be used
to guess the mime content type of a file. If failure, then rejects with a
RuntimeException.

```php
React\mime_content_type("/tmp/file.png", $loop)
    ->then(function(string $type) {
        // Do whatever you need with the found mime type
    });
```

This is a non-blocking action and equals the regular PHP function
[mime_content_type](https://www.php.net/manual/en/function.mime-content-type.php).

# Adapters

In order to be able to use DriftPHP and your whole domain in a non-blocking way,
all your infrastructure operation must be done by using ReactPHP libraries. This
framework, and taking the advantage of Symfony bundle architecture, offers you a
set of components to make you configure these clients in a more easy and
structured way.

## Redis adapter

[![CircleCI](https://circleci.com/gh/driftphp/redis-bundle.svg?style=svg)](https://circleci.com/gh/driftphp/redis-bundle)

This is a simple adapter for Redis on top of ReactPHP and DriftPHP. Following
the same structure that is followed in the Symfony ecosystem, you can use this
package as a Bundle, only usable under DriftPHP Framework.

### Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/redis-bundle) from Github.

```bash
composer require drift/redis-bundle
```

### Configure

This package will allow you to configure all your Redis async clients, taking
care of duplicity and the loop integration. Once your package is required by
composer, add the bundle in the kernel and change your `services.yaml`
configuration file to defined the clients

```yaml
redis:
    clients:
        users:
            host: 127.0.0.
            port: 6379
            database: /users
            password: secret
            protocol: rediss://
            idle: 0.5
            timeout: 10.0
            preload: true
        orders:
            host: 127.0.0.1
            database: /orders
```

Only host is required. All the other values are optional.

### Usage

Once you have your clients created, you can inject them in your services by
using the name of the client in your dependency injection arguments array

```yaml
a_service:
    class: My\Service
    arguments:
        - "@redis.users_client"
        - "@redis.orders_client"
```

You can use Autowiring as well in the bundle, by using the name of the client
and using it as a named parameter

```php
use Clue\React\Redis\Client;

public function __construct(
    Client $usersClient,
    Client $ordersClient
)
```

> By enabling the preload flag, you're creating the connected client before the
> first request is handled. This means that as soon as you handle the first one,
> the client will be already ready to use.

## Mysql adapter

[![CircleCI](https://circleci.com/gh/driftphp/mysql-bundle.svg?style=svg)](https://circleci.com/gh/driftphp/mysql-bundle)

This is a simple adapter for Mysql on top of ReactPHP and DriftPHP. Following
the same structure that is followed in the Symfony ecosystem, you can use this
package as a Bundle, only usable under DriftPHP Framework.

### Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/mysql-bundle) from Github.

```bash
composer require drift/mysql-bundle
```

### Configure

This package will allow you to configure all your Mysql async clients, taking
care of duplicity and the loop integration. Once your package is required by
composer, add the bundle in the kernel and change your `services.yaml`
configuration file to defined the connections

```yaml
mysql:
    connections:
        users:
            host: 127.0.0.1
            port: 3306
            user: root
            password: root
            database: users
        orders:
            host: 127.0.0.1
            user: root
            password: root
            database: orders
```

All parameters are required, but the port. This one is `3306` by default.

### Usage

Once you have your connections created, you can inject them in your services by
using the name of the connection in your dependency injection arguments array

```yaml
a_service:
    class: My\Service
    arguments:
        - "@mysql.users_connection"
        - "@mysql.orders_connection"
```

You can use Autowiring as well in the bundle, by using the name of the 
connection and using it as a named parameter

```php
use React\MySQL\ConnectionInterface;
use React\MySQL\Io\LazyConnection;

public function __construct(
    ConnectionInterface $usersConnection,
    LazyConnection $ordersConnection
)
```

## AMQP adapter

[![CircleCI](https://circleci.com/gh/driftphp/amqp-bundle.svg?style=svg)](https://circleci.com/gh/driftphp/amqp-bundle)

This is a simple adapter for AMQP on top of ReactPHP and DriftPHP. Following
the same structure that is followed in the Symfony ecosystem, you can use this
package as a Bundle, only usable under DriftPHP Framework.

> AMQP is the protocol that uses RabbitMQ

### Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/amqp-bundle) from Github.

```bash
composer require drift/amqp-bundle
```

### Configure

This package will allow you to configure all your Mysql async clients, taking
care of duplicity and the loop integration. Once your package is required by
composer, add the bundle in the kernel and change your `services.yaml`
configuration file to defined the connections

```yaml
amqp:
    clients:
        commands:
            host: 127.0.0.1
            port: 5672
            vhost: /
            user: guest
            password: guest
            preload: true
        events:
            host: 127.0.0.1
```

All parameters are required, but the port. This one is `3306` by default.

### Usage

Once you have your connections created, you can inject them in your services by
using the name of the connection in your dependency injection arguments array

```yaml
a_service:
    class: My\Service
    arguments:
        - "@amqp.commands_client"
```

You can use Autowiring as well in the bundle, by using the name of the 
connection and using it as a named parameter

```php
use Bunny\AbstractClient;
use Bunny\Async\Client;
use Bunny\Channel;

public function __construct(
    AbstractClient $commandsClient,
    Client $eventsClient,
    Channel $commandsChannel
)
```

> By enabling the preload flag, you're creating the connected client before the
> first request is handled. This means that as soon as you handle the first one,
> the client will be already ready to use.

## Twig adapter

[![CircleCI](https://circleci.com/gh/driftphp/twig-bundle.svg?style=svg)](https://circleci.com/gh/driftphp/twig-bundle)

This is a simple adapter for Twig on top of ReactPHP and DriftPHP. Following
the same structure that is followed in the Symfony ecosystem, you can use this
package as a Bundle, only usable under DriftPHP Framework.

This package will work for you as a simple bridge between your DriftPHP and Twig
as a templating engine. Of course, the behavior will be exactly the same than
working on top of other frameworks, like Symfony, but some small changes have
been introduced here in order to be non-blocking.

### Installation

You can install the package by using composer, or getting the 
[source code](https://github.com/driftphp/twig-bundle) from Github.

```bash
composer require drift/twig-bundle
```

### Usage

You have two different ways of using Twig in DriftPHP. Both ways are pretty
similar, but could be discussed about the grade of coupling that we want between
our layers.

The first one is the regular one. We can inject Twig as a dependency, and having
the path of our template, we can easily render our view. You can check here a
small example about this strategy. As you can see, your controller turns
dependent on a view reference and on the complete Twig engine, which can make
sense somehow.

```php
/**
 * Class ViewValuesController
 */
class ViewValuesController
{
    private $twig;

    /**
     * PutValueController constructor.
     *
     * @param Environment $twig
     */
    public function __construct(Environment $twig)
    {
        $this->twig = $twig;
    }

    /**
     * Invoke
     *
     * @param Request $request
     *
     * @return PromiseInterface
     */
    public function __invoke(Request $request)
    {
        $template = $this
            ->twig
            ->load('redis/view_values.twig');

        return new Response(
            $template->render([
                'values' => $values
            ]),
            200
    }
}
```

On the other hand, you can make your controller **only** a Twig coupled part by
implementing a small interface, which will force you defining this template path
without having to inject the Environment. In that case, you can return an array
and the bundle will render properly the template.

```php
use Drift\Twig\Controller\RenderableController;

/**
 * Class ViewValueController
 */
class ViewValueController implements RenderableController
{
    /**
     * Invoke
     *
     * @param Request $request
     *
     * @return array
     */
    public function __invoke(Request $request) : array
    {
        return [
            'key' => 'value'
        ];
    }

    /**
     * Get render template
     *
     * @return string
     */
    public static function getTemplatePath(): string
    {
        return 'redis/view_value.twig';
    }
}
```

If you follow that strategy, then you will be able to return promises instead of
simple values. In that case, the bundle will wait until all promises are
properly fulfilled and will render the template.

```php
use Drift\Twig\Controller\RenderableController;

/**
 * Class ViewValueController
 */
class ViewValueController implements RenderableController
{
    /**
     * Invoke
     *
     * @param Request $request
     *
     * @return array
     */
    public function __invoke(Request $request) : array
    {
        return [
            'key' => (new FulfilledPromise())
                ->then(function() {
                    return 'value';
                });
        ];
    }

    /**
     * Get render template
     *
     * @return string
     */
    public static function getTemplatePath(): string
    {
        return 'redis/view_value.twig';
    }
}
