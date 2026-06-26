# Framework-X Integrations

Use this reference for databases, authentication, sessions, child processes,
filesystem access, queues, streaming, and templates.

## Contents

- [Database](#database)
- [Authentication](#authentication)
- [Sessions](#sessions)
- [Filesystem](#filesystem)
- [Child Processes](#child-processes)
- [Queueing](#queueing)
- [Streaming](#streaming)
- [Templates](#templates)
- [Sources](#sources)

## Database

Prefer async database adapters for Framework-X, especially with the built-in
server. Supported ReactPHP ecosystem adapters include MySQL, Postgres, SQLite,
Redis, ClickHouse, and others.

Default guidance:

- Use async database APIs instead of PDO/Doctrine in hot async routes.
- Use placeholders and parameter arrays for SQL query parameters.
- Put database logic in repositories/services once it grows beyond trivial
  queries.
- Use DI/container bindings for connection interfaces.
- For PHP 8.1+, use fibers and `await()` for ergonomic code.
- Use promises for low-level repository APIs when explicit async return types
  are preferred.

Container binding:

```php
$container = new FrameworkX\Container([
    React\MySQL\ConnectionInterface::class => function (string $MYSQL_URI) {
        return (new React\MySQL\Factory())->createLazyConnection($MYSQL_URI);
    },
]);
```

Parameterized query:

```php
$result = await($db->query(
    'SELECT title FROM book WHERE isbn = ?',
    [$isbn]
));
```

Do not build SQL by concatenating user input. Parameterized statements are
faster for many adapters, safer against injection, and easier to read.

### Connection Lifetime

Traditional stacks use shared-nothing execution: request state and database
connections are cleaned up after each request.

The built-in server is long-running:

- Creating a connection per request mimics shared-nothing behavior but loses
  connection reuse benefits.
- Reusing one connection improves setup cost but serializes queries and can
  suffer head-of-line blocking.
- A connection pool is the preferred compromise when available.

The official DBAL and connection pool material is marked as feature preview, so
verify current package support before depending on it.

### Blocking Database APIs

Legacy blocking APIs can work, but they block the event loop. If no async
adapter exists, isolate blocking work in child processes or keep the blocking
operation outside high-concurrency paths.

## Authentication

The official auth integration page is an early draft. Use these defaults:

- Implement authentication as HTTP middleware.
- Use request attributes to pass authenticated user/principal data downstream.
- Return early with `401 Unauthorized` or `403 Forbidden` when needed.
- HTTP Basic auth is straightforward for simple/internal tools.
- JWT/OAuth are possible but application-specific.
- Credentials often require database access or sessions.

Middleware shape:

```php
class AuthMiddleware
{
    public function __invoke(ServerRequestInterface $request, callable $next)
    {
        $token = $request->getHeaderLine('Authorization');

        if ($token === '') {
            return Response::plaintext("Unauthorized\n")
                ->withStatus(Response::STATUS_UNAUTHORIZED);
        }

        return $next($request->withAttribute('user', $this->authenticate($token)));
    }
}
```

## Sessions

The official sessions page is an early draft. Use middleware for session
handling:

- Read the session identifier from a cookie/header.
- Load session data from storage before the controller.
- Attach session/user data to request attributes.
- Persist changes and attach `Set-Cookie` after the downstream handler returns.
- Make response-side session middleware async-aware if controllers can return
  promises or coroutines.

Use database or Redis storage when sessions must survive process restarts or
scale across multiple server processes/containers.

## Filesystem

Avoid blocking filesystem calls in hot async routes:

- `fopen()`, `file_get_contents()`, `readfile()`, and related calls block.
- A few small blocking calls may be acceptable when measured and explicit.
- Prefer async filesystem APIs where available.
- Use child processes for blocking filesystem workloads.
- For object storage, consider async S3 libraries such as `clue/reactphp-s3`.

Also remember the request upload/body parsing limit from the request docs when
designing uploads.

## Child Processes

Use child processes to keep blocking or CPU-heavy work out of the event loop.

- Communicate over child process I/O.
- Treat child processes as isolated workers, not shared-memory threads.
- Use `reactphp/child-process` for low-level process control.
- Consider `clue/reactphp-pq` for wrapping blocking functions as non-blocking
  promises.

Good candidates:

- Legacy blocking database/client libraries.
- Image/document conversion.
- Large filesystem scans.
- CPU-heavy operations.

## Queueing

Use queues to offload work from request/response paths to background workers.

Options mentioned by the official docs:

- BunnyPHP for AMQP/RabbitMQ.
- `clue/reactphp-redis` for Redis blocking lists and streams.
- Experimental STOMP support for RabbitMQ, Apollo, ActiveMQ, and similar.

Agent defaults:

- Return `202 Accepted` or a job identifier when enqueueing asynchronous work.
- Keep queue payloads small and serializable.
- Make handlers idempotent where retries are possible.
- Persist job state when the client needs progress or result lookup.

## Streaming

Use streaming when data is large or arrives over time:

- Streaming downloads avoid holding full file contents in memory.
- Server-Sent Events/EventSource are supported out of the box for live,
  one-way updates.
- WebSockets can be integrated with Ratchet for bidirectional communication.
- Consider SSE before WebSockets when the client only needs server-to-client
  events.

## Templates

Any PHP template engine can be used:

- Twig
- Handlebars
- Mustache
- Native PHP rendering

Templates are most relevant for HTML responses. Watch filesystem access:
template files loaded from disk can block, so cache compiled templates or use
async/isolated loading where appropriate.

```php
return React\Http\Message\Response::html($html);
```

## Sources

- https://github.com/clue/framework-x/blob/main/docs/integrations/authentication.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/child-processes.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/database.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/filesystem.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/queueing.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/sessions.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/streaming.md
- https://github.com/clue/framework-x/blob/main/docs/integrations/templates.md
