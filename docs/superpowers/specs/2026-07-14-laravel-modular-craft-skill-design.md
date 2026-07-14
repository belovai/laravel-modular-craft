# laravel-modular-craft — Claude Code Skill Design

## Purpose

A standalone, project-independent Claude Code skill that teaches Claude the
author's modular Laravel architecture conventions, so it can be dropped into
any Laravel project and Claude will follow the same structure when creating
or reviewing modules, controllers, actions, services, DTOs/value objects,
and tests.

This is a **guidance skill**, not a code generator: no stub/template files
are scaffolded automatically. The skill's job is to make Claude apply the
right structure and naming when it writes Laravel code by hand, and to
recognize violations during review.

Source of truth for the conventions below: real code from the author's
`Gixer` project (`modules/Auth`, `modules/User`, `modules/Permission`,
`modules/Probe`) and `Medicare/mex` project (`modules/Invoice`), reviewed
and confirmed/corrected during design.

## Non-goals

- No artisan command / generator that scaffolds files.
- No opinion on frontend, database schema design, or deployment.
- No enforcement tooling (no PHP-CS-Fixer/Rector config) — this is a skill
  for Claude's own code generation and review judgment, not a static
  analysis pipeline. (A future skill could add that; out of scope here.)

## Repo structure

```
laravel-modular-craft/
├── SKILL.md
├── references/
│   ├── architecture.md
│   ├── controllers.md
│   ├── actions-and-services.md
│   ├── dto-and-value-objects.md
│   ├── testing.md
│   └── php-style.md
├── LICENSE
└── README.md
```

Content language: **English** (the skill should be usable by anyone, on any
project, not just the author).

`SKILL.md` stays short: frontmatter description (for accurate auto-trigger),
a compact decision tree, and pointers into `references/*.md` for depth.
Reference files are loaded by Claude only when the relevant topic is
actually in play, following the same progressive-disclosure pattern used by
other Claude Code skills (e.g. `superpowers:using-superpowers` referencing
platform-specific docs only when needed).

## SKILL.md content plan

**Frontmatter description** (drives auto-invocation): something like —
"Use when creating, structuring, or reviewing Laravel code that follows a
modular architecture — module folders, slim controllers, Action classes,
DTOs/Value Objects, Services. Trigger when writing a new Laravel
controller, action, service, DTO/VO, module scaffold, or module-level
test."

**Body**: a decision tree covering the core routing questions, each
pointing to the relevant reference file:

- New feature area → module folder layout → `architecture.md`
- New HTTP endpoint, not CRUD → invoke (`__invoke`) controller → `controllers.md`
- New HTTP endpoint, CRUD → resource controller → `controllers.md`
- Logic that mutates data, or that must be callable identically from HTTP
  controller / artisan command / queued job / MCP server → Action → `actions-and-services.md`
- Complex logic shared across multiple Actions, or wrapping an external
  vendor/API → Service → `actions-and-services.md`
- Passing/returning more than one related value instead of an array →
  DTO or VO → `dto-and-value-objects.md`
- Writing or placing a test → `testing.md`
- General PHP class conventions → `php-style.md`

## references/architecture.md

Module folder layout (module = top-level feature area, e.g. `Auth`, `User`,
`Permission`):

```
modules/{ModuleName}/
├── Actions/
├── Console/Commands/          (if the module has artisan commands)
├── Contracts/                 (interfaces, e.g. for Services)
├── Controllers/
├── Database/{Factories,Migrations,Seeders}/
├── DataTransferObjects/
├── Enums/
├── Exceptions/
├── Http/Middleware/           (if the module has its own middleware)
├── Models/
├── Providers/
├── Requests/
├── Resources/
├── Routes/
├── Services/
├── Support/                   (small module-local helpers that don't fit elsewhere)
├── Tests/{Feature,Unit}/
├── Traits/
└── ValueObjects/
```

Rule: **subfolder names are always spelled out**, following Laravel's own
convention (`Contracts`, `Traits`, `Support`, `Exceptions`,
`DataTransferObjects`, `ValueObjects`) — never abbreviated (`Dto`, `VOs`,
`Ctrl`, etc.). Only create the subfolders a module actually needs; don't
pre-create empty ones.

**Provider registration**: every module has exactly one
`{Module}ModuleServiceProvider` in `Providers/`, extending a shared
abstract `App\Providers\ModuleServiceProvider` base class. The base class
exposes a `key()` helper (derived from the provider's class name) used for
things like permission-namespace registration. The module provider's
`boot()` method is where everything module-scoped gets wired up:
`loadMigrationsFrom()`, `loadRoutesFrom()`, permission/gate registration,
and any other module bootstrapping. Nothing module-specific should leak
into the app-level `AppServiceProvider`.

Include the real (genericized) `AccessToken` VO and
`UserModuleServiceProvider` boot() example from Gixer as a concrete
illustration.

## references/controllers.md

Two controller shapes only — **no other controller style is allowed**:

1. **Invoke (action) controller** — for anything that isn't standard CRUD
   (login, purge, sync, any single verb-shaped operation). `final class`
   with a single `__invoke()` method. Named after the action, e.g.
   `LoginController`.
2. **CRUD (resource) controller** — for standard resource operations
   (`index`, `show`, `store`, `update`, `destroy`). `final class`, one
   method per operation.

**Slim controller rule**: a controller method may contain simple,
one-off, non-reusable reads directly (e.g. `index`/`show` doing a plain
Eloquent query) — but *any* domain logic or data mutation must be
delegated to an Action. This isn't about CRUD-vs-not; it's about reuse:
the same Action must be callable identically from an HTTP controller, an
artisan command, a queued job, or an MCP server, with the same inputs. If
a "read" stops being a one-liner (filtering, authorization-shaped
assembly, cross-model composition), it graduates to an Action too.

Controllers own: request validation (via `Requests/`), authorization
(`Gate::authorize()`), calling the Action, and shaping the HTTP response
(via `Resources/` + a shared response-formatting trait — see
`php-style.md`). Controllers do not own business logic.

Include the real (genericized) `LoginController` and `UserController`
examples from Gixer, annotated to show which methods are "plain read" vs
"delegates to Action" and why.

## references/actions-and-services.md

**Action** — the default home for domain logic and data mutation.

- One class, one job: named `VerbNoun` (`CreateUser`, `PurgeTokens`,
  `GenerateToken`), `final class`, single public `handle()` method.
- Exists specifically so the same operation is reusable, with identical
  inputs/output, from multiple entry points: HTTP controller, artisan
  command, queued job, MCP server tool. If logic will only ever be called
  from one controller method and involves no real domain rule, a plain
  controller one-liner is fine (see `controllers.md`) — but default to an
  Action once there's any real logic.
- Actions can depend on Services and other Actions via constructor
  injection.

**Service** — for logic that's more than one Action needs, or that wraps
an external vendor/API.

- `final readonly class`, constructor-promoted dependencies (e.g. base
  URL, credentials, an injected HTTP client).
- When a Service wraps an external system, define a `Contracts/` interface
  for it and bind the concrete implementation in the module's provider —
  this keeps the Action layer testable/mockable against the interface.
- Do not use a Service just to avoid a slightly longer Action — only reach
  for one when logic is genuinely shared or externally-vendored.

Include the real (genericized) `CreateUser` Action and
`RabbitMqManagementService` + its Contract from Gixer/Probe as examples.

## references/dto-and-value-objects.md

Both live under module subfolders spelled out in full:
`DataTransferObjects/` and `ValueObjects/` — never `Dto`/`VO` abbreviated
in the namespace or folder.

**Value Object (VO)** — a small, cohesive, self-meaningful value.

- `final readonly class`, constructor property promotion only, public
  readonly properties, no `fromArray`/`toArray` machinery.
- Use when the value is meaningful standing alone (e.g. `AccessToken`,
  `PlainToken`, `Money`).

**DTO** — for carrying a larger bag of related data across a boundary
instead of returning/passing a bare array.

- Can have `fromArray()` / `toArray()` (or implement Laravel's
  `Arrayable`) when the data genuinely needs to move in and out of array
  form (e.g. request payload → DTO → external API call).
- Do **not** bake JSON serialization concerns into every DTO by default.
  If a DTO needs `JsonSerializable` or similar, that behavior should come
  from a shared trait (e.g. a `Serializable`-style trait), not be
  hand-rolled per DTO. This is a nice-to-have, not a required part of the
  DTO convention — don't add it speculatively.

**The core rule driving both**: never return an array just because a
method needs to return 2+ related values. If it's a single meaningful
value → VO. If it's a larger, more request/response-shaped bag of related
fields → DTO.

Include the real `AccessToken` VO (Gixer) and `CustomerDTO`
(`fromArray`/`toArray`) (Medicare/mex) examples, genericized.

## references/testing.md

- Tests live **inside the module**: `modules/{Module}/Tests/Feature/` and
  `modules/{Module}/Tests/Unit/`. Never centralize module tests under a
  top-level `tests/` directory.
- Pest syntax (`describe`/`it`), not PHPUnit class-based tests.
- Feature tests exercise the HTTP surface end-to-end (route → controller →
  action → response), naming the file after the operation
  (`CreateUserTest`, `DestroyUserTest`).
- Unit tests target a single Action/Service/VO/DTO in isolation when the
  behavior warrants it (not required for every trivial class).
- Include the real (genericized) `CreateUserTest` Pest example from Gixer.

## references/php-style.md

General class conventions to apply across the module system (not just
Actions/Services/VOs/DTOs):

- `declare(strict_types=1);` at the top of every PHP file.
- `final class` by default; only remove `final` when inheritance is a
  deliberate, documented design choice.
- `readonly` properties wherever the value doesn't need to change after
  construction (common on VOs, DTOs, Services).
- Constructor property promotion instead of manually declared properties +
  assignment.
- Enums instead of magic strings/constants for closed sets of values
  (e.g. `UserStatus::Active`), including custom helper methods on the enum
  when useful (e.g. `notEquals()`).
- API response formatting is extracted into a shared trait (the
  convention, not a specific trait name/implementation — the author's
  current example is `ApiResponses`, but that trait itself is
  app-level/cross-module, not part of the module-authoring pattern this
  skill teaches).

## README.md content plan

- What this skill is and who it's for (Laravel projects using this
  specific modular architecture).
- How to install it into another project as a Claude Code skill (copy
  into the project's `.claude/skills/` or reference as a plugin —
  finalize exact mechanism during implementation, matching whatever
  Claude Code's current skill-loading convention is at build time).
- One-paragraph summary of the architecture so a human skimming the repo
  also understands it without opening `SKILL.md`.

## Success criteria

- Dropping this skill into a fresh Laravel project and asking Claude to
  "add a new module for X" or "add an endpoint that does Y" produces code
  matching the structure above without further correction.
- Claude correctly distinguishes Action vs Service vs DTO vs VO cases when
  asked to review or write code, matching the decision rules above.
- The skill triggers automatically on relevant Laravel work without being
  explicitly named.
