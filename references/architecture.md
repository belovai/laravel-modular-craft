# Architecture: modules and providers

## Module folder layout

A module is a top-level feature area (`Auth`, `User`, `Permission`,
`Invoice` — not a technical layer). Every module lives under
`modules/{ModuleName}/`:

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

**Rule: subfolder names are always spelled out**, following Laravel's own
convention (`Contracts`, `Traits`, `Support`, `Exceptions`,
`DataTransferObjects`, `ValueObjects`) — never abbreviated (`Dto`, `VOs`,
`Ctrl`, etc.).

Only create the subfolders a module actually needs. Don't pre-create empty
ones "for completeness" — a module with no artisan commands has no
`Console/Commands/` folder at all.

## Provider registration

Every module has **exactly one** `{Module}ModuleServiceProvider` in its
`Providers/` folder, extending a shared abstract
`App\Providers\ModuleServiceProvider` base class. The base class exposes a
`key()` helper, derived from the provider's class name, used for things
like permission-namespace registration:

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Str;

abstract class ModuleServiceProvider extends ServiceProvider
{
    public function key(): string
    {
        $name = Str::before(class_basename(static::class), 'ModuleServiceProvider');

        return Str::snake($name);
    }
}
```

The module provider's `boot()` method is where everything module-scoped
gets wired up: `loadMigrationsFrom()`, `loadRoutesFrom()`,
permission/gate registration, and any other module bootstrapping.
**Nothing module-specific leaks into the app-level `AppServiceProvider`.**

```php
<?php

declare(strict_types=1);

namespace Modules\User\Providers;

use App\Providers\ModuleServiceProvider;
use Illuminate\Support\Facades\Gate;

final class UserModuleServiceProvider extends ModuleServiceProvider
{
    public function boot(): void
    {
        $this->loadMigrationsFrom(__DIR__ . '/../Database/Migrations');
        $this->loadRoutesFrom(__DIR__ . '/../Routes/api.php');

        Gate::define(
            "{$this->key()}.manage",
            fn ($user) => $user->hasPermissionTo("{$this->key()}.manage"),
        );
    }
}
```

The provider must be explicitly registered in the application's `bootstrap/providers.php` — it is not auto-discovered. This file returns an array of provider class names, one entry per module:

```
return [
    App\Providers\AppServiceProvider::class,
    Modules\User\Providers\UserModuleServiceProvider::class,
    // ...one entry per module
];
```

Each module's provider is added to this array when the module is created. Without this registration, Laravel never instantiates the provider, and its `boot()` method never runs.

## Example Value Object referenced from a provider

Providers, Actions, and Controllers in the `User` module all deal with
tokens through a small Value Object rather than a raw string — see
`references/dto-and-value-objects.md` for when to reach for a VO versus a
DTO:

```php
<?php

declare(strict_types=1);

namespace Modules\User\ValueObjects;

final readonly class AccessToken
{
    public function __construct(
        public string $plainText,
        public string $hashed,
        public \DateTimeImmutable $expiresAt,
    ) {
    }

    public function isExpired(): bool
    {
        return $this->expiresAt < new \DateTimeImmutable();
    }
}
```
