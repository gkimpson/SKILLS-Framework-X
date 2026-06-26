---
name: framework-x-best-practices
description: Use when building, structuring, deploying, or testing a Framework-X application, or when deciding between architectural patterns (controllers vs closures, deployment stacks, service composition)
---

# Framework-X Best Practices

## Overview

Framework-X is a modern PHP web framework for event-loop driven applications using ReactPHP. It runs anywhere (traditional stacks, standalone) but shines with its **built-in async web server**. Best practices differ from traditional PHP frameworks because of async/Promise handling and the flexibility to choose deployment.

**Core principle:** Start simple (closures), evolve to controllers when complexity demands, structure around services and dependency injection, test from day one.

---

## Project Structure

### Progression: Closures → Controllers → Services

**When to use closures** (prototype/MVP phase):
- Fewer than 8 routes total
- No shared logic between handlers
- Business logic is trivial (return data, simple transformations)

```php
$app->get('/', function () {
    return Response::plaintext("Hello World!\n");
});
```

**When to refactor to controllers** (once you have >8 routes OR shared logic):
- Multiple routes exist
- Handlers exceed 20 lines
- Logic is testable (validation, calculations, API calls)
- Code appears duplicated across handlers

```php
$app->get('/', HelloController::class);
$app->get('/users/{name}', UserController::class);
```

**Controller structure (minimal):**

```php
namespace Acme\Todo;

use Psr\Http\Message\ServerRequestInterface;
use React\Http\Message\Response;

class UserController
{
    public function __invoke(ServerRequestInterface $request)
    {
        $name = $request->getAttribute('name');
        return Response::plaintext("Hello $name!\n");
    }
}
```

**Key requirements:**
- Use **PSR-4 autoloading** in `composer.json`:
  ```json
  {
    "autoload": {
      "psr-4": {
        "Acme\\Todo\\": "src/"
      }
    }
  }
  ```
  Then run `composer dump-autoload` once.
- Controllers implement `__invoke` (invokable classes) or named methods
- Accept `ServerRequestInterface` from the framework (dependency injection)
- Return `ResponseInterface` or `PromiseInterface` (for async operations)

---

## Composition: Services & Dependency Injection

### When to extract a Service Layer

Extract business logic to a Service when:
1. Logic is used by multiple controllers
2. Logic handles external dependencies (HTTP clients, databases, caches)
3. You need to test logic in isolation

**Don't extract prematurely.** If one controller owns the logic, keep it simple for now.

### Basic Service Pattern

```php
// src/Service/EAClubsService.php
namespace Acme\Service;

use React\Http\Browser;
use React\Promise\PromiseInterface;

class EAClubsService
{
    private Browser $http;

    public function __construct(Browser $http)
    {
        $this->http = $http;
    }

    public function fetchClubs(int $limit): PromiseInterface
    {
        return $this->http
            ->get('https://api.example.com/clubs?limit=' . $limit)
            ->then(fn($response) => json_decode($response->getBody(), true));
    }
}
```

### Using the Container (Dependency Injection)

Framework-X includes a container for registering services:

```php
// public/index.php
$app = new FrameworkX\App();

// Register the service
$app->get(EAClubsService::class, function () {
    $http = new Browser();
    return new EAClubsService($http);
});

// Handler receives service from container
$app->get('/clubs', function (EAClubsService $service) {
    return $service->fetchClubs(10)
        ->then(fn($clubs) => Response::json($clubs))
        ->catch(fn($error) => Response::json(['error' => $error->getMessage()], 500));
});
```

**Key pattern:** Services return `PromiseInterface`, handlers compose Promises naturally.

---

## Testing

### Unit Test Structure (PHPUnit)

```bash
composer require --dev phpunit/phpunit
vendor/bin/phpunit tests
```

**Test pattern:**
1. Create request object
2. Pass to controller/handler
3. Assert response

```php
// tests/UserControllerTest.php
namespace Acme\Tests\Todo;

use Acme\Todo\UserController;
use PHPUnit\Framework\TestCase;
use React\Http\Message\ServerRequest;

class UserControllerTest extends TestCase
{
    public function testControllerReturnsValidResponse()
    {
        $request = new ServerRequest('GET', 'http://example.com/users/Alice');
        $request = $request->withAttribute('name', 'Alice');

        $controller = new UserController();
        $response = $controller($request);

        $this->assertEquals(200, $response->getStatusCode());
        $this->assertEquals("Hello Alice!\n", (string)$response->getBody());
    }
}
```

### Testing Async Operations (Services with Promises)

```php
// tests/EAClubsServiceTest.php
namespace Acme\Tests\Service;

use Acme\Service\EAClubsService;
use PHPUnit\Framework\TestCase;
use React\Http\Browser;
use React\Promise\Promise;

class EAClubsServiceTest extends TestCase
{
    public function testFetchClubsReturnsData()
    {
        // Mock the HTTP client
        $http = $this->createMock(Browser::class);
        $http->method('get')->willReturn(
            new Promise(function ($resolve) {
                $response = $this->createMock(ResponseInterface::class);
                $response->method('getBody')->willReturn(json_encode([
                    ['id' => 1, 'name' => 'Club A'],
                    ['id' => 2, 'name' => 'Club B'],
                ]));
                $resolve($response);
            })
        );

        $service = new EAClubsService($http);
        
        $service->fetchClubs(10)->then(function ($clubs) {
            $this->assertCount(2, $clubs);
            $this->assertEquals('Club A', $clubs[0]['name']);
        });
    }
}
```

**Testing priorities (not 100% coverage):**
- Business logic (validation, calculations, state transitions)
- External API integrations (mock external calls)
- Error handling (what happens on failure?)
- Skip: UI rendering, logging, formatting

### Running Tests with Docker

```bash
# Run all tests
docker compose exec app vendor/bin/phpunit

# Run tests with coverage report
docker compose exec app vendor/bin/phpunit --coverage-text

# Run specific test file
docker compose exec app vendor/bin/phpunit tests/UserControllerTest.php
```

**Usage with watch mode:**
- Start development: `docker compose up --build --watch`
- In another terminal: `docker compose exec app vendor/bin/phpunit`
- Tests run against live code synced into container

---

## Deployment

### Option 1: Built-In Web Server (Quick Start, Development)

```bash
php public/index.php
# or run on specific port
php -S 0.0.0.0:8080 public/index.php
```

**When to use:** Local development, prototyping, MVP testing.

**Advantages:** Zero configuration, natural async concurrency, immediate feedback.

---

### Option 2: Docker Compose with Watch (Recommended for Production)

Use Docker with `docker compose --build --watch` for modern deployments. This gives you containerized consistency, live file sync, and easy scaling.

**docker-compose.yml:**

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      MYSQL_URI: "${MYSQL_USER}:${MYSQL_PASSWORD}@mysql:3306/${MYSQL_DATABASE}"
    depends_on:
      mysql:
        condition: service_healthy
    develop:
      watch:
        - action: sync+restart
          path: ./src
          target: /app/src
        - action: sync+restart
          path: ./public
          target: /app/public
        - action: rebuild
          path: ./composer.json

  mysql:
    image: mysql:8.4
    ports:
      - "3307:3306"
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - ./schema.sql:/docker-entrypoint-initdb.d/schema.sql:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 5s
      timeout: 5s
      retries: 10
```

**Dockerfile:**

```dockerfile
FROM php:8.2-cli-alpine

RUN docker-php-ext-install pdo_mysql

WORKDIR /app

COPY composer.json composer.lock ./
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer && \
    composer install --no-scripts

COPY src ./src
COPY public ./public

EXPOSE 8080

CMD ["php", "public/index.php"]
```

**Usage:**

```bash
# Development (live reload on file changes)
docker compose --build --watch up

# Production (standard up)
docker compose up -d
```

**Watch configuration explained:**
- `sync+restart`: Copy file changes from host into container, restart the service
- `rebuild`: Rebuild image when `composer.json` changes (new dependencies)

**Advantages:**
- Consistent environment (dev = staging = prod)
- Live file sync for rapid iteration
- Easy dependency management (MySQL, Redis, etc. as services)
- Zero-downtime deploys (orchestrate container restarts)
- Multi-container orchestration built-in

---

### Option 3: Nginx Reverse Proxy + Multiple Built-In Servers (High Concurrency, Single Host)

For maximum concurrency on a single machine, run multiple Framework-X processes (one per CPU core) behind Nginx:

**docker-compose.yml (no services section shown, assume similar to Option 2):**

```yaml
services:
  app1:
    build: .
    ports:
      - "8001:8080"
    environment:
      MYSQL_URI: "..."
    depends_on:
      mysql:
        condition: service_healthy

  app2:
    build: .
    ports:
      - "8002:8080"
    # same config
    
  app3:
    build: .
    ports:
      - "8003:8080"
    # same config

  app4:
    build: .
    ports:
      - "8004:8080"
    # same config

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app1
      - app2
      - app3
      - app4
```

**nginx.conf (snippet):**

```nginx
upstream frameworkx {
    server app1:8080;
    server app2:8080;
    server app3:8080;
    server app4:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://frameworkx;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**When to use:** Production with high concurrency needs, multi-core utilization, zero-downtime deploys.

**Advantages:** Full async concurrency per container, leverages all CPU cores, load balancing.

---

### Option 4: Caddy (Alternative to Nginx, Easier Setup)

Caddy auto-provisions HTTPS and has simpler reverse proxy configuration:

```caddyfile
example.com {
    reverse_proxy localhost:8001 localhost:8002 localhost:8003 localhost:8004 {
        policy random_choose 2
    }
}
```

**When to use:** If auto HTTPS provisioning and minimal config are priorities.

---

### Option 5: Nginx + PHP-FPM (Traditional Stack—Not Recommended)

**Don't use this for Framework-X in production.** Nginx + PHP-FPM assumes blocking request/response. Framework-X's async event loop sits idle waiting for FPM to respond—you lose all concurrency benefits.

**If legacy infrastructure forces this:**

```nginx
server {
    root /var/www/html/public;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass localhost:9000;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

**Reality:** This works but wastes async advantages. Plan to migrate toward Docker or Nginx reverse proxy.

---

## Architecture Decision Tree

```
Do I have multiple routes?
├─ NO → Closures (prototype phase)
└─ YES → Controllers
    
    Does logic repeat across handlers?
    ├─ NO → Keep in controllers
    └─ YES → Extract to Service
        
        Does service call external APIs/DB?
        ├─ NO → Keep method on Service
        └─ YES → Use Container (DI) for HTTP client injection
        
Ready for production deployment?
├─ Local development only → Built-in server
├─ Production, single host → Docker Compose (Option 2)
├─ Production, multi-core → Docker + Nginx reverse proxy (Option 3)
└─ Legacy infrastructure → Nginx + PHP-FPM (but migrate if possible)
```

---

## Common Mistakes

| Mistake | Reality |
|---------|---------|
| Premature service extraction | Extract when logic repeats or external deps appear, not on day 1 |
| Using Nginx + PHP-FPM for Framework-X | Wastes async. Use reverse proxy + built-in servers or Docker instead. |
| Skipping tests "because we're async" | Async code is *harder* to test manually. Tests are essential. |
| Handlers mixing HTTP logic with business logic | Extract business to services; handlers orchestrate. |
| Not running `composer dump-autoload` after adding namespace | Autoloader doesn't find classes. Always dump when adding PSR-4 paths. |
| Using callbacks instead of Promise chains | `->then()` chains are clearer than nested callbacks. Use promises naturally. |
| Not setting up watch in docker compose | Manual rebuilds on every change. Use `--build --watch` for hot reload. |

---

## Quick Reference

| Pattern | When | Why |
|---------|------|-----|
| **Closures** | <8 routes, MVP | Fast to start, no structure overhead |
| **Controllers** | >8 routes, testable logic | Organized, testable, reusable |
| **Services** | Shared logic, external deps | Encapsulation, DI, testability |
| **Middleware** | Cross-cutting concerns | Auth, logging, rate-limiting globally |
| **Built-in server** | Local dev, testing | Simple, efficient, async-native |
| **Docker Compose** | Production, any scale | Consistent env, live reload, easy scaling |
| **Nginx reverse proxy** | High concurrency, single host | Multi-core, zero-downtime deploys |

---

## Connected Documentation

- **[Framework-X Docs](https://github.com/clue/framework-x)** — Official reference
- **Controllers best practices** — https://github.com/clue/framework-x/blob/main/docs/best-practices/controllers.md
- **Deployment guide** — https://github.com/clue/framework-x/blob/main/docs/best-practices/deployment.md
- **Testing guide** — https://github.com/clue/framework-x/blob/main/docs/best-practices/testing.md
- **Container/DI** — https://github.com/clue/framework-x/blob/main/docs/api/app.md
- **Docker Compose Watch** — https://docs.docker.com/compose/file-watch/
