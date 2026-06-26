# Framework-X Architecture and Testing

Use this reference when structuring a Framework-X app, refactoring routes into
controllers/services/repositories, configuring the container, or adding tests.

## Contents

- [Philosophy for Structure](#philosophy-for-structure)
- [Closures to Controllers](#closures-to-controllers)
- [PSR-4 Autoloading](#psr-4-autoloading)
- [Container and Dependency Injection](#container-and-dependency-injection)
- [Database-Oriented Class Structure](#database-oriented-class-structure)
- [Testing](#testing)
- [Sources](#sources)

## Philosophy for Structure

- Start simple: closure routes are acceptable for quickstarts, prototypes, and
  tiny apps.
- Move to controller classes once the application leaves the prototype phase or
  route files become hard to scan.
- Extract services/repositories when logic repeats, talks to external systems,
  or needs isolated tests.
- Accept small duplication over premature abstractions.
- Let the business domain drive structure; Framework-X does not enforce one
  architecture.

## Closures to Controllers

Closure route:

```php
$app->get('/users/{name}', function (Psr\Http\Message\ServerRequestInterface $request) {
    return React\Http\Message\Response::plaintext(
        'Hello ' . $request->getAttribute('name') . "!\n"
    );
});
```

Controller route:

```php
$app->get('/users/{name}', Acme\Todo\UserController::class);
```

Controller:

```php
namespace Acme\Todo;

use Psr\Http\Message\ServerRequestInterface;
use React\Http\Message\Response;

class UserController
{
    public function __invoke(ServerRequestInterface $request)
    {
        return Response::plaintext(
            'Hello ' . $request->getAttribute('name') . "!\n"
        );
    }
}
```

## PSR-4 Autoloading

Add namespace mapping once:

```json
{
  "autoload": {
    "psr-4": {
      "Acme\\Todo\\": "src/"
    }
  }
}
```

Then run:

```bash
composer dump-autoload
```

After this, new classes under `src/` are autoloaded normally.

## Container and Dependency Injection

Framework-X includes a lightweight DI container with autowiring:

- Route handlers may be registered by class name.
- Class names must be autoloadable.
- Constructor dependencies with class types are autowired.
- Optional constructor arguments are omitted unless explicitly configured.
- Interfaces, primitive scalars, specific instances, and alternate classes need
  explicit configuration.

```php
$container = new FrameworkX\Container([
    React\MySQL\ConnectionInterface::class => function (string $MYSQL_URI) {
        return (new React\MySQL\Factory())->createLazyConnection($MYSQL_URI);
    },
]);

$app = new FrameworkX\App($container);
$app->get('/book/{isbn}', Acme\Todo\BookLookupController::class);
```

Factory functions can receive other container entries automatically:

```php
$container = new FrameworkX\Container([
    Acme\Todo\UserController::class => function (React\Http\Browser $browser) {
        return new Acme\Todo\UserController($browser, 42);
    },
]);
```

Scalar values and environment variables:

- Container variables share the same map as class entries; avoid collisions by
  using unique names.
- Process environment variables are available automatically as uppercase
  container variables.
- Environment values are strings; cast or accept union types when needed.
- You may override Framework-X variables such as `X_LISTEN` via the container.

```php
$container = new FrameworkX\Container([
    'debug' => false,
    'hostname' => fn (): string => gethostname(),
    'X_LISTEN' => fn (string $X_LISTEN = '127.0.0.1:8080') => $X_LISTEN,
]);
```

Interface mapping:

```php
$container = new FrameworkX\Container([
    React\Cache\CacheInterface::class => React\Cache\ArrayCache::class,
]);
```

External containers: if a container implements `Psr\Container\ContainerInterface`,
adapt it with `new FrameworkX\Container($container)` before passing it to
`new FrameworkX\App(...)`.

## Database-Oriented Class Structure

Recommended starting structure:

```text
acme/
├── public/
│   └── index.php
├── src/
│   ├── Book.php
│   ├── BookRepository.php
│   └── BookLookupController.php
├── tests/
│   ├── BookTest.php
│   ├── BookRepositoryTest.php
│   └── BookLookupControllerTest.php
├── vendor/
├── composer.json
└── composer.lock
```

Responsibilities:

- Entity/value object: domain data and invariants.
- Repository/service: database/API access and business operations.
- Controller: request attributes/body to domain call, domain result to response.

As the app grows, either group by domain:

```text
src/
├── Book/
│   ├── Book.php
│   ├── BookRepository.php
│   └── BookLookupController.php
└── User/
    ├── User.php
    ├── UserRepository.php
    └── UserLookupController.php
```

Or group by functionality:

```text
src/
├── Controllers/
├── Entities/
└── Repositories/
```

Prefer grouping by domain when the application has distinct business areas.

## Testing

Install PHPUnit:

```bash
composer require --dev phpunit/phpunit
```

Controller test pattern:

1. Create a `React\Http\Message\ServerRequest`.
2. Add attributes/headers/body/params needed for the case.
3. Invoke the controller directly.
4. Assert `ResponseInterface`, status code, headers, and body.

```php
namespace Acme\Tests\Todo;

use Acme\Todo\UserController;
use PHPUnit\Framework\TestCase;
use Psr\Http\Message\ResponseInterface;
use React\Http\Message\ServerRequest;

class UserControllerTest extends TestCase
{
    public function testControllerReturnsValidResponse()
    {
        $request = new ServerRequest('GET', 'http://example.com/users/Alice');
        $request = $request->withAttribute('name', 'Alice');

        $response = (new UserController())($request);

        $this->assertInstanceOf(ResponseInterface::class, $response);
        $this->assertSame(200, $response->getStatusCode());
        $this->assertSame("Hello Alice!\n", (string) $response->getBody());
    }
}
```

Testing priorities:

- Controller behavior for key routes and request attributes.
- Status codes and response bodies for success, not found, validation errors,
  and failures.
- Repository/service logic with database/API dependencies mocked or replaced.
- Async code return shape and resolved value.
- Middleware request mutations and response transformations.

For promises, resolve the promise in the test with the project's existing
ReactPHP testing utilities or use fibers/`await()` if already used in the
project test suite. Do not assert only that a promise object exists; assert the
eventual response/data.

Run tests:

```bash
vendor/bin/phpunit tests
```

## Sources

- https://github.com/clue/framework-x/blob/main/docs/best-practices/controllers.md
- https://github.com/clue/framework-x/blob/main/docs/best-practices/testing.md
- https://github.com/clue/framework-x/blob/main/docs/getting-started/community.md
- https://github.com/clue/framework-x/blob/main/docs/getting-started/philosophy.md
- https://github.com/clue/framework-x/blob/main/docs/getting-started/quickstart.md
- https://github.com/clue/framework-x/blob/main/docs/README.md
