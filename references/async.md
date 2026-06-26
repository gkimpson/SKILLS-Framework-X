# Framework-X Async Patterns

Use this reference when handlers, services, middleware, databases, HTTP clients,
filesystem work, queues, child processes, or streams involve asynchronous APIs.

## Contents

- [Decision Guide](#decision-guide)
- [Fibers](#fibers)
- [Promises](#promises)
- [Coroutines](#coroutines)
- [Middleware and Async](#middleware-and-async)
- [Failure Handling](#failure-handling)
- [Sources](#sources)

## Decision Guide

- Use **fibers** when PHP 8.1+ is available and you want async APIs to look like
  synchronous code. This is the preferred ergonomic default for new code.
- Use **promises** when building low-level async APIs, composing explicit
  concurrency, or targeting maximum performance/control.
- Use **coroutines** when the project is on older PHP or already uses
  Generator-based `yield` flows.
- Do not mix styles casually inside a single abstraction. Pick one style for an
  API boundary and document the return type.

Framework-X can consume handlers that return:

- `Psr\Http\Message\ResponseInterface`
- `React\Promise\PromiseInterface` resolving to a response
- `Generator`/coroutine resolving to a response

## Fibers

Fibers let code `await()` promises without changing the method's public return
type into a promise or generator. Framework-X supports fibers through
`reactphp/async`.

```php
use function React\Async\await;

$app->get('/book', function () use ($db) {
    $result = await($db->query('SELECT COUNT(*) AS count FROM book'));

    return React\Http\Message\Response::plaintext(
        'Found ' . $result->resultRows[0]['count'] . " books\n"
    );
});
```

Production guidance:

- Prefer PHP 8.1+ for production fiber use.
- Compatibility mode on older PHP can stop/restart the loop under concurrency
  and is suitable only for development or low-concurrency fast handlers.
- Catch exceptions around `await()` when the async operation can fail.

## Promises

Promises are the underlying building block used throughout Framework-X and
ReactPHP. They are efficient and explicit, but every caller must understand that
the method is async.

```php
use React\Promise\PromiseInterface;

class BookRepository
{
    public function findBook(string $isbn): PromiseInterface
    {
        return $this->db->query(
            'SELECT title FROM book WHERE isbn = ?',
            [$isbn]
        )->then(function (React\MySQL\QueryResult $result) {
            return $result->resultRows === []
                ? null
                : new Book($result->resultRows[0]['title']);
        });
    }
}
```

Controller using a promise-returning repository:

```php
public function __invoke(ServerRequestInterface $request): PromiseInterface
{
    return $this->repository
        ->findBook($request->getAttribute('isbn'))
        ->then(function (?Book $book) {
            if ($book === null) {
                return Response::plaintext("Book not found\n")
                    ->withStatus(Response::STATUS_NOT_FOUND);
            }

            return Response::plaintext($book->title);
        });
}
```

## Coroutines

Coroutines use `yield` to unwrap promises. Any function containing `yield`
returns a `Generator`, so callers must handle generators or use `yield from`.

```php
class BookRepository
{
    public function findBook(string $isbn): \Generator
    {
        $result = yield $this->db->query(
            'SELECT title FROM book WHERE isbn = ?',
            [$isbn]
        );

        return $result->resultRows === []
            ? null
            : new Book($result->resultRows[0]['title']);
    }
}
```

Controller consuming the coroutine:

```php
public function __invoke(ServerRequestInterface $request): \Generator
{
    $book = yield from $this->repository->findBook($request->getAttribute('isbn'));

    if ($book === null) {
        return Response::plaintext("Book not found\n")
            ->withStatus(Response::STATUS_NOT_FOUND);
    }

    return Response::plaintext($book->title);
}
```

## Middleware and Async

Request middleware that only modifies the request can return `$next($request)`
without caring whether the next handler is sync or async.

Response middleware must handle all possible downstream return shapes:

- `PromiseInterface`: use `then()` and modify the resolved response.
- `Generator`: return a wrapping coroutine with `yield from`.
- `ResponseInterface`: modify directly.

See [Core API](core-api.md#middleware) for the canonical async-aware response
middleware pattern.

## Failure Handling

- Attach promise rejection handlers with `then($onFulfilled, $onRejected)` or
  `otherwise()` when the route should produce a domain-specific response.
- Let unexpected errors bubble to Framework-X's `ErrorHandler`.
- With fibers, wrap `await()` in `try/catch` for expected failures.
- With coroutines, exceptions thrown while resolving yielded promises surface
  when the coroutine resumes.

## Sources

- https://github.com/clue/framework-x/blob/main/docs/async/fibers.md
- https://github.com/clue/framework-x/blob/main/docs/async/coroutines.md
- https://github.com/clue/framework-x/blob/main/docs/async/promises.md
