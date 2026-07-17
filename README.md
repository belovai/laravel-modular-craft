# laravel-modular-craft

A Claude Code skill that teaches a modular Laravel architecture: module
folders, slim controllers, Action classes, Services, DTOs/Value Objects,
and module-local Pest tests. Drop it into any Laravel project that uses
(or wants to adopt) this structure, and Claude will follow it when writing
or reviewing controllers, actions, services, DTOs/VOs, and tests — without
needing to be re-taught the conventions each time.

This is a **guidance skill**, not a generator: it doesn't scaffold files
or run an artisan command. It shapes how Claude writes and reviews code by
hand.

## Who this is for

Laravel projects organized around feature modules (`modules/{Module}/`
containing their own Controllers, Actions, Models, routes, migrations, and
tests) rather than the framework's default flat `app/` structure. If your
project uses a different architecture, this skill's rules won't apply and
shouldn't be installed.

## Installing

Clone the repo directly into your project's Claude Code skills directory:

```bash
git clone https://github.com/belovai/laravel-modular-craft.git \
  .claude/skills/laravel-modular-craft
```

Or, if you prefer a one-off copy without the git history:

```bash
curl -L https://github.com/belovai/laravel-modular-craft/archive/refs/heads/main.tar.gz \
  | tar -xz --strip-components=1 -C .claude/skills/laravel-modular-craft
```

Claude Code discovers any `SKILL.md` under `.claude/skills/*/`
automatically — no further configuration needed.

## The architecture, in one paragraph

Each feature area is a module under `modules/{ModuleName}/`, with its own
spelled-out subfolders (`Actions/`, `Controllers/`,
`DataTransferObjects/`, `ValueObjects/`, `Services/`,
`Tests/{Feature,Unit}/`, and so on) and a single
`{Module}ModuleServiceProvider` that wires up its migrations, routes, and
gates. Controllers come in exactly two shapes — invoke controllers for
single-verb operations, resource controllers for CRUD — and stay slim:
they validate, authorize, call an Action, and shape the response, never
holding business logic themselves. Any logic that mutates data, or that
needs to run identically from a controller, an artisan command, a queued
job, or an MCP server, lives in a single-purpose Action; logic shared
across Actions or wrapping an external system lives in a Service instead.
Multi-value returns are a Value Object (a small, self-meaningful value) or
a DTO (a larger, request/response-shaped bag of fields) — never a bare
array. Tests live inside the module they cover, written in Pest's
`describe`/`it` style.

See `SKILL.md` for the full decision tree and `references/*.md` for each
topic in depth.

## License

MIT — see [LICENSE](LICENSE).
