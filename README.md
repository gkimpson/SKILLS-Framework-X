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
- Dependency injection and container usage
- Promise-based async service composition
- PHPUnit testing patterns for handlers and async services
- Docker Compose deployments with watch mode
- Reverse proxy deployments with Nginx or Caddy
- When to avoid traditional Nginx + PHP-FPM setups
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
└── SKILL.md
```

`SKILL.md` contains the actual agent instructions and examples.

## Installation

Clone or copy this repository into your local skills directory:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/YOUR_USERNAME/framework-x-best-practices.git \
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

Update `SKILL.md` when Framework-X, ReactPHP, Docker Compose, or deployment
best practices change. Keep examples small, runnable, and focused on the
decision they demonstrate.

Before publishing changes:

```bash
git diff --check
```
