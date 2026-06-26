# Framework-X Deployment

Use this reference when choosing or implementing a deployment model.

## Contents

- [Deployment Decision](#deployment-decision)
- [Built-In Web Server](#built-in-web-server)
- [Memory Limits](#memory-limits)
- [File Descriptor Limits](#file-descriptor-limits)
- [Systemd](#systemd)
- [Nginx Reverse Proxy](#nginx-reverse-proxy)
- [Caddy Reverse Proxy](#caddy-reverse-proxy)
- [Traditional PHP-FPM Stacks](#traditional-php-fpm-stacks)
- [Docker](#docker)
- [Operational Checks](#operational-checks)
- [Sources](#sources)

## Deployment Decision

- Use PHP's development server only for local testing:
  `php -S 0.0.0.0:8080 public/index.php`.
- Use traditional PHP stacks when existing hosting requires it. Framework-X
  supports Nginx/Caddy/Apache with PHP-FPM or related stacks.
- Prefer the built-in Framework-X web server for async-native production. Run it
  behind Nginx, Caddy, systemd, or Docker.
- Use Docker containers for reproducible environments and cloud/serverless
  portability.

Traditional stacks work, but shared-nothing PHP execution gives up much of the
benefit of a long-running async server. Treat PHP-FPM support as compatibility,
not the preferred high-concurrency architecture.

## Built-In Web Server

Run:

```bash
php public/index.php
```

Default listen address is `127.0.0.1:8080`, which is appropriate behind a
reverse proxy. Configure with `X_LISTEN`:

```bash
X_LISTEN=127.0.0.1:8081 php public/index.php
X_LISTEN=0.0.0.0:8080 php public/index.php
```

Avoid exposing the built-in server directly to the public unless the deployment
context deliberately handles TLS, static assets, limits, and operational
concerns elsewhere.

## Memory Limits

The built-in server handles many concurrent requests in one long-running PHP
process. Default PHP memory limits may be too low for concurrency spikes.

For production CLI/server use, consider:

```ini
memory_limit = -1
```

Use application-level limits and operational monitoring rather than assuming
single-request PHP memory behavior.

## File Descriptor Limits

High concurrency can exceed default FD limits such as `1024`.

```bash
ulimit -n 100000
```

Also install a scalable event loop extension when raising FD limits:

- `ext-ev` recommended
- `ext-event`
- `ext-uv` beta

Without one, `stream_select()` can hit `FD_SETSIZE` limits and cause warnings or
high CPU.

## Systemd

Use a process manager for production restarts:

```ini
[Unit]
Description=ACME server

[Service]
ExecStart=/usr/bin/php /home/alice/projects/acme/public/index.php
User=alice
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl enable acme.service
sudo systemctl start acme.service
sudo systemctl restart acme.service
```

Adjust paths, user, PHP binary, environment variables, and restart policy for
the target host.

## Nginx Reverse Proxy

Recommended pattern with built-in server:

```nginx
server {
    location / {
        location ~* \.php$ {
            try_files /dev/null @x;
        }
        root /home/alice/projects/acme/public;
        try_files $uri @x;
    }

    location @x {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header Connection "";
    }

    location ~ \.htaccess$ {
        try_files /dev/null @x;
    }
}
```

Minimal proxy-only form:

```nginx
server {
    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header Connection "";
    }
}
```

Add TLS, static asset policy, logging, rate limits, and trusted proxy handling
for real production.

## Caddy Reverse Proxy

```caddyfile
example.com {
    root * /var/www/html/public;

    @static {
        file {path} {path}/
        not path *.php
    }
    handle @static {
        rewrite * {http.matchers.file.relative}
        file_server
    }

    handle {
        reverse_proxy localhost:8080
    }
}
```

Caddy is useful when automatic HTTPS and simpler proxy configuration are
priorities.

## Traditional PHP-FPM Stacks

Framework-X runs behind Nginx, Caddy, Apache, PHP-FPM, FastCGI, and shared
hosting-style stacks. Use the standard `public/` docroot and rewrite dynamic
requests to `public/index.php`.

Nginx + PHP-FPM shape:

```nginx
server {
    root /home/alice/projects/acme/public;
    index index.php index.html;

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

Apache needs `DocumentRoot` at `public/` and rewrite rules in `.htaccess`.
Remember to forward the authorization header when auth depends on it.

## Docker

Development/simple image:

```dockerfile
# syntax=docker/dockerfile:1
FROM php:8.5-cli

WORKDIR /app/
COPY public/ public/
COPY vendor/ vendor/

ENV X_LISTEN=0.0.0.0:8080
EXPOSE 8080

ENTRYPOINT ["php", "public/index.php"]
```

Production image pattern:

```dockerfile
# syntax=docker/dockerfile:1
FROM composer:2 AS build

WORKDIR /app/
COPY composer.json composer.lock ./
RUN composer install --no-dev --ignore-platform-reqs --optimize-autoloader

FROM php:8.5-alpine

RUN apk --no-cache add ${PHPIZE_DEPS} libev linux-headers \
    && pecl install ev \
    && docker-php-ext-enable ev \
    && docker-php-ext-install sockets \
    && apk del ${PHPIZE_DEPS} linux-headers \
    && echo "memory_limit = -1" >> "$PHP_INI_DIR/conf.d/acme.ini"

WORKDIR /app/
COPY public/ public/
COPY src/ src/
COPY --from=build /app/vendor/ vendor/

ENV X_LISTEN=0.0.0.0:8080
EXPOSE 8080

USER nobody:nobody
ENTRYPOINT ["php", "public/index.php"]
```

Run:

```bash
docker build -t acme .
docker run -it --rm -p 8080:8080 acme
docker run -d --ulimit nofile=100000 -p 8080:8080 acme
```

## Operational Checks

- Confirm the app responds through the final public URL, not only localhost.
- Confirm static files are served by the proxy when configured.
- Confirm dynamic routes reach Framework-X.
- Confirm access logs record the intended client IP behind proxies.
- Confirm memory and FD limits match expected concurrency.
- Confirm service/container restarts after code deploys.
- Confirm environment variables are available to the container or service.

## Sources

- https://github.com/clue/framework-x/blob/main/docs/best-practices/deployment.md
- https://github.com/clue/framework-x/blob/main/docs/getting-started/quickstart.md
