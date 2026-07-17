---
name: laravel-modular-craft
description: Use when creating, structuring, or reviewing Laravel code that follows a modular architecture — module folders, slim controllers, Action classes, DTOs/Value Objects, Services. Trigger when writing a new Laravel controller, action, service, DTO/VO, module scaffold, or module-level test.
---

# Laravel Modular Craft

## Overview

Conventions for a modular Laravel architecture: one folder per feature area
under `modules/`, slim controllers, single-purpose Action classes, Services
reserved for shared/external logic, and DTOs/Value Objects instead of bare
arrays. This is a guidance skill — it does not scaffold files. It teaches
Claude the structure and naming to apply by hand when writing or reviewing
Laravel code, in any project that uses this architecture.

## Decision tree

- Starting a new feature area? → module folder layout → `references/architecture.md`
- New HTTP endpoint, not standard CRUD (login, purge, sync, any single
  verb-shaped operation)? → invoke (`__invoke`) controller →
  `references/controllers.md`
- New HTTP endpoint, standard CRUD (`index`/`show`/`store`/`update`/`destroy`)?
  → resource controller → `references/controllers.md`
- Logic that mutates data, or that must be callable identically from an HTTP
  controller, an artisan command, a queued job, or an MCP server? → Action →
  `references/actions-and-services.md`
- Complex logic shared across multiple Actions, or wrapping an external
  vendor/API? → Service → `references/actions-and-services.md`
- Returning or passing 2+ related values instead of a bare array? → DTO or
  Value Object → `references/dto-and-value-objects.md`
- Writing or placing a test? → `references/testing.md`
- General PHP class conventions (strict types, final, readonly, enums)? →
  `references/php-style.md`

## Core rule

Controllers own request validation, authorization, calling the Action, and
shaping the HTTP response. They never own business logic — see the
slim-controller rule in `references/controllers.md` for exactly where that
line falls.
