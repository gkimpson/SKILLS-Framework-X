---
name: framework-x-best-practices
description: Use when building, structuring, testing, deploying, reviewing, or integrating a Framework-X PHP application. Covers Framework-X App/routing, middleware, PSR-7 requests and responses, async promises/coroutines/fibers, controller and container architecture, PHPUnit testing, production deployment, and integrations including authentication, sessions, database, filesystem, queueing, streaming, templates, and child processes.
---

# Framework-X Best Practices

Use this skill to make practical implementation decisions for
[Framework-X](https://github.com/clue/framework-x), the ReactPHP-based PHP web
framework. Favor the framework's philosophy: make easy things easy, keep hard
things possible, start with small working code, and introduce structure only
when it clarifies the application.

## Source Coverage

This skill summarizes the official Framework-X docs listed below. When a task
needs detail, load the matching reference file instead of relying only on this
top-level guide.

- Core API: `App`, middleware, request, response
- Async: fibers, coroutines, promises
- Best practices: controllers, deployment, testing
- Getting started: quickstart, philosophy, community, docs README
- Integrations: authentication, child processes, database, filesystem,
  queueing, sessions, streaming, templates

## Reference Routing

Read the relevant reference before implementing non-trivial work:

- [Core API](references/core-api.md): routes, redirects, request data,
  responses, caching, error/access logging, middleware chains.
- [Async](references/async.md): choosing and composing fibers, coroutines, and
  promises.
- [Architecture and Testing](references/architecture-testing.md): closures vs
  controllers, PSR-4 autoloading, DI container, repositories, PHPUnit tests.
- [Deployment](references/deployment.md): PHP-FPM compatibility, built-in web
  server, reverse proxies, systemd, Docker, memory and FD limits.
- [Integrations](references/integrations.md): database, auth, sessions,
  filesystem, child processes, queues, streaming, templates.

## Default Workflow

1. Identify the execution model first:
   - Traditional stack: shared-nothing PHP execution through PHP-FPM, Apache, or
     similar. Works, but does not use the async server model.
   - Built-in server: long-running event-loop process. Prefer this for
     async-native production, usually behind Nginx, Caddy, or Docker.
2. Start with closure routes for quick prototypes and tiny apps.
3. Move to invokable controller classes when routes or request logic grow.
4. Put business/data access logic in services or repositories once it repeats,
   requires dependencies, or needs isolated tests.
5. Use the built-in container or explicit constructor injection to wire
   dependencies.
6. Use async APIs for I/O. Prefer fibers on PHP 8.1+, promises for low-level
   async APIs or explicit concurrency, and coroutines for older PHP/codebases
   already using `yield`.
7. Add tests around controller behavior, request attributes, response status
   and body, database/service logic, and async failure paths.
8. For production, prefer the built-in server behind a reverse proxy or in a
   container. Tune memory and file descriptor limits for high concurrency.

## Design Rules

- Do not over-abstract early. A little duplication is acceptable if an
  abstraction would hide the domain.
- Keep HTTP handling at the edge. Controllers translate requests into domain
  calls and domain results into responses.
- Keep long-running server state intentional. With the built-in server,
  services and connections may live across requests.
- Avoid blocking I/O in hot paths. If no async adapter exists, isolate blocking
  work in a child process or keep it small and explicit.
- Use PSR standards where possible: PSR-7 request/response objects and PSR-11
  containers.
- Implement cross-cutting behavior as middleware: auth, sessions, logging,
  trusted proxy handling, cache headers, and response transformations.
- Make response middleware async-aware if downstream handlers can return
  promises or coroutines.
- Prefer placeholders and query parameters over hand-built strings for routes
  and SQL.

## Quick Patterns

Minimal app:

```php
<?php

require __DIR__ . '/../vendor/autoload.php';

$app = new FrameworkX\App();

$app->get('/', function () {
    return React\Http\Message\Response::plaintext("Hello world!\n");
});

$app->run();
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

Container binding:

```php
$container = new FrameworkX\Container([
    React\MySQL\ConnectionInterface::class => function (string $MYSQL_URI) {
        return (new React\MySQL\Factory())->createLazyConnection($MYSQL_URI);
    },
]);

$app = new FrameworkX\App($container);
```

Async database with fibers:

```php
use function React\Async\await;

$app->get('/book/{isbn}', function (Psr\Http\Message\ServerRequestInterface $request) use ($db) {
    $result = await($db->query(
        'SELECT title FROM book WHERE isbn = ?',
        [$request->getAttribute('isbn')]
    ));

    if (count($result->resultRows) === 0) {
        return React\Http\Message\Response::plaintext("Book not found\n")
            ->withStatus(React\Http\Message\Response::STATUS_NOT_FOUND);
    }

    return React\Http\Message\Response::plaintext($result->resultRows[0]['title']);
});
```

## Common Mistakes

| Mistake | Better approach |
| --- | --- |
| Treating Framework-X like only PHP-FPM | It runs there, but the built-in server unlocks async concurrency. |
| Modifying async responses as if they are always `ResponseInterface` | Handle `PromiseInterface` and `Generator` in response middleware. |
| Building SQL with string concatenation | Use query placeholders and parameter arrays. |
| Forgetting `composer dump-autoload` after adding PSR-4 config | Add PSR-4 once, dump autoload, then add classes normally. |
| Storing per-request state in long-lived services | Keep state local to the request or reset it explicitly. |
| Using blocking filesystem/database/process calls in hot routes | Prefer async adapters or child processes. |
| Returning invalid values from controllers | Return `ResponseInterface`, `PromiseInterface`, or `Generator`/coroutine that resolves to a response. |

## Official Documentation Links

- https://github.com/clue/framework-x/blob/main/docs/api/app.md
- https://github.com/clue/framework-x/blob/main/docs/api/middleware.md
- https://github.com/clue/framework-x/blob/main/docs/api/request.md
- https://github.com/clue/framework-x/blob/main/docs/api/response.md
- https://github.com/clue/framework-x/blob/main/docs/async/fibers.md
- https://github.com/clue/framework-x/blob/main/docs/async/coroutines.md
- https://github.com/clue/framework-x/blob/main/docs/async/promises.md
- https://github.com/clue/framework-x/blob/main/docs/best-practices/controllers.md
- https://github.com/clue/framework-x/blob/main/docs/best-practices/deployment.md
- https://github.com/clue/framework-x/blob/main/docs/best-practices/testing.md
- https://github.com/clue/framework-x/blob/main/docs/getting-started/community.md
- https://github.com/clue/framework-x/blob/main/docs/getting-started/philosophy.md
- https://github.com/clue/framework-x/blob/main/docs/getting-started/quickstart.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/authentication.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/child-processes.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/database.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/filesystem.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/queueing.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/sessions.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/streaming.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/templates.md
- https://github.com/clue/framework-x/blob/main/docs/README.md
