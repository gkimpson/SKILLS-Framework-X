# Framework-X Core API

Use this reference for App routing, PSR-7 requests and responses, middleware,
redirects, access logs, error handling, and HTTP response details.

## Contents

- [App and Routing](#app-and-routing)
- [Requests](#requests)
- [Responses](#responses)
- [Redirects and Caching](#redirects-and-caching)
- [Output Buffering](#output-buffering)
- [Error Handling](#error-handling)
- [Access Logs](#access-logs)
- [Middleware](#middleware)
- [Sources](#sources)

## App and Routing

- Create the application with `new FrameworkX\App()` and call `$app->run()` from
  `public/index.php`.
- Register route handlers with `$app->get()`, `head()`, `post()`, `put()`,
  `patch()`, `delete()`, `options()`, `map()`, and `any()`.
- URI placeholders such as `/user/{id}` become request attributes accessible
  through `$request->getAttribute('id')`.
- `GET` routes match `HEAD` requests by default unless a more explicit `HEAD`
  route exists. Framework-X discards the response body for `HEAD`.
- Unknown paths return `404 Not Found`; unsupported methods return
  `405 Method Not Allowed`.
- Use `$app->redirect($from, $to)` for simple `302 Found` redirects, or pass a
  custom `3xx` status such as `Response::STATUS_MOVED_PERMANENTLY`.

```php
$app->get('/user/{id}', UserController::class);
$app->map(['GET', 'POST'], '/user/{id}', UserController::class);
$app->any('/health', HealthController::class);
$app->redirect('/old', '/new', React\Http\Message\Response::STATUS_MOVED_PERMANENTLY);
```

## Requests

Framework-X uses PSR-7 request objects. Type-hint
`Psr\Http\Message\ServerRequestInterface`; do not depend on the concrete
request class unless constructing requests in tests.

- Attributes: route placeholders and middleware-provided data.
- JSON body: `(string) $request->getBody()` then `json_decode()`. Validate
  `Content-Type: application/json` when semantics matter.
- Form data: `$request->getParsedBody()` returns arrays similar to `$_POST`.
- Uploads: `$request->getUploadedFiles()` returns `UploadedFileInterface`
  objects similar to `$_FILES`.
- Headers: `$request->getHeaderLine('Header-Name')`.
- Server params: `$request->getServerParams()`, similar to `$_SERVER`.

Important upload/body limit: request bodies are currently limited to 64 KiB in
the official docs. Larger uploads can show as empty body/no uploaded files, so
do not design large upload flows around default request parsing.

```php
use Psr\Http\Message\ServerRequestInterface;
use React\Http\Message\Response;

$app->post('/users', function (ServerRequestInterface $request) {
    $data = json_decode((string) $request->getBody(), true);
    $name = $data['name'] ?? 'anonymous';

    return Response::json(['message' => "Hello $name"]);
});
```

## Responses

Framework-X accepts PSR-7 responses. Prefer `React\Http\Message\Response`
helpers unless manual control is needed.

- `Response::plaintext($body)` for plain text.
- `Response::json($data)` for JSON with default `200 OK` and
  `Content-Type: application/json`.
- `Response::html($html)` for HTML with default `200 OK` and
  `Content-Type: text/html; charset=utf-8`.
- Use `withStatus()` for status codes and `withHeader()` for response headers.
- Use `Response::STATUS_*` constants instead of magic numbers when readable.
- For custom JSON flags or streaming/body control, construct
  `new React\Http\Message\Response($status, $headers, $body)`.

```php
return Response::json($data)
    ->withStatus(Response::STATUS_CREATED)
    ->withHeader('Cache-Control', 'no-store');
```

## Redirects and Caching

- Redirects are normal `3xx` responses with a `Location` header; use
  `$app->redirect()` for simple route redirects.
- Use `Cache-Control` for freshness policy.
- Use `ETag` plus `If-None-Match` or `Last-Modified` plus `If-Modified-Since`
  to return `304 Not Modified` without a body.
- Format HTTP dates in GMT, for example `gmdate('D, d M Y H:i:s T', $ts)`.

Prefer ETags when you do not have a trustworthy modification timestamp or do
not want to expose one.

## Output Buffering

Avoid `echo`, `print`, `var_dump`, `readfile`, and similar output-buffer writes
inside handlers. Prefer functions that return strings, then put the string into
a response. If integrating legacy code, wrap it carefully:

```php
ob_start();
try {
    legacy_render();
    $body = ob_get_clean();
} catch (Throwable $e) {
    ob_end_clean();
    throw $e;
}

return Response::html($body);
```

## Error Handling

- Controllers must return a response, promise, or coroutine/generator resolving
  to a response.
- Throwing `Throwable` or returning an invalid value becomes a `500 Internal
  Server Error`.
- Framework-X automatically adds a default `ErrorHandler` if one is not
  explicitly passed first.
- The default error message hides internal details from clients.
- For custom error behavior, catch expected exceptions in application code or
  implement custom middleware.

```php
$app = new FrameworkX\App(
    FrameworkX\ErrorHandler::class
);
```

## Access Logs

With the built-in web server, Framework-X logs requests/responses to `STDOUT`
by default through `AccessLogHandler`.

- If passed explicitly, `AccessLogHandler` must be followed by `ErrorHandler`.
- It is intended as global middleware, not route-level middleware.
- Configure a log file by binding an absolute path in the container or passing
  it to `new AccessLogHandler($path)`.
- Disable logs by pointing to `/dev/null` on Unix or `nul` on Windows.
- Behind a reverse proxy, use trusted proxy middleware before the access logger
  if you want logs to contain the original client IP from `X-Forwarded-For`.

## Middleware

Middleware wraps handlers. It receives
`ServerRequestInterface $request, callable $next` and may:

- return a response early;
- modify the incoming request;
- call `$next($request)`;
- modify the outgoing response;
- return a synchronous response, a promise, or a coroutine.

Use middleware classes for production code. Inline middleware is fine for
experiments.

```php
class AdminMiddleware
{
    public function __invoke(ServerRequestInterface $request, callable $next)
    {
        if (($request->getServerParams()['REMOTE_ADDR'] ?? null) === '127.0.0.1') {
            $request = $request->withAttribute('admin', true);
        }

        return $next($request);
    }
}
```

Request-only middleware usually works with synchronous and async handlers
without special logic, because it returns `$next($request)` unchanged. Response
middleware must be async-aware if downstream handlers may return promises or
coroutines:

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use React\Promise\PromiseInterface;

class ContentTypeMiddleware
{
    public function __invoke(ServerRequestInterface $request, callable $next)
    {
        $response = $next($request);

        if ($response instanceof PromiseInterface) {
            return $response->then(fn (ResponseInterface $response) => $this->handle($response));
        }

        if ($response instanceof \Generator) {
            return (fn () => $this->handle(yield from $response))();
        }

        return $this->handle($response);
    }

    private function handle(ResponseInterface $response): ResponseInterface
    {
        return $response->withHeader('Content-Type', 'text/plain');
    }
}
```

Global middleware is passed to `new FrameworkX\App(...)`. It runs for all
registered routes and unroutable requests. Global middleware runs before
route-level middleware.

## Sources

- https://github.com/clue/framework-x/blob/main/docs/api/app.md
- https://github.com/clue/framework-x/blob/main/docs/api/middleware.md
- https://github.com/clue/framework-x/blob/main/docs/api/request.md
- https://github.com/clue/framework-x/blob/main/docs/api/response.md
