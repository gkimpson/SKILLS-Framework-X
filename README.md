# Framework-X Best Practices Skill

A reusable agent skill for building, structuring, testing, and deploying
[Framework-X](https://github.com/clue/framework-x) applications.

Framework-X is an event-loop driven PHP framework built on ReactPHP. This skill
captures practical guidance for using it well: start small with route closures,
move to controllers when the application grows, extract services when logic or
dependencies need isolation, and deploy in a way that preserves the framework's
async benefits.

## What This Skill Covers

- Project structure for closures, controllers, and services
- Framework-X App routing, redirects, error handling, and access logs
- Middleware, including async-aware response middleware
- PSR-7 request and response handling
- Dependency injection and container usage
- Fibers, coroutines, and promise-based async composition
- PHPUnit testing patterns for handlers and async services
- Built-in server, PHP-FPM, systemd, Docker, Nginx, and Caddy deployments
- Integrations for auth, sessions, database, filesystem, queueing, streaming,
  templates, and child processes
- Common Framework-X architecture mistakes

## When To Use It

Use this skill when an agent is helping with:

- Creating a new Framework-X application
- Refactoring route closures into controllers
- Deciding when to introduce a service layer
- Testing controllers, handlers, or async service code
- Choosing a deployment approach for Framework-X
- Reviewing a Framework-X project for architecture issues

## Contents

```text
.
├── README.md
├── SKILL.md
└── references/
    ├── architecture-testing.md
    ├── async.md
    ├── core-api.md
    ├── deployment.md
    └── integrations.md
```

`SKILL.md` contains the trigger-level guidance and points agents to the focused
reference files when a task needs more detail.

## Installation

Clone or copy this repository into your local skills directory:

```bash
mkdir -p ~/.claude/skills
git clone <repository-url> \
  ~/.claude/skills/framework-x-best-practices
```

If you are using another agent runtime that supports `SKILL.md` files, install
the repository in that runtime's skill directory instead.

## Usage

Once installed, ask your agent for Framework-X help. For example:

```text
Use the framework-x-best-practices skill to review this app structure.
```

```text
Help me deploy this Framework-X application with Docker Compose.
```

```text
Refactor these Framework-X routes into controllers and services.
```

The skill is designed to guide practical implementation decisions, not just
describe Framework-X concepts.

## Maintenance

Update `SKILL.md` and the relevant file under `references/` when Framework-X,
ReactPHP, Docker, or integration best practices change. Keep examples small,
runnable, and focused on the decision they demonstrate.

Before publishing changes:

```bash
git diff --check
```
