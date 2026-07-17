# Controllers

Two controller shapes only — **no other controller style is allowed**.

## 1. Invoke (action) controller

For anything that isn't standard CRUD: login, purge, sync, any single
verb-shaped operation. `final class` with a single `__invoke()` method,
named after the action it performs (`LoginController`,
`PurgeTokensController`).

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Controllers;

use App\Http\Traits\ApiResponses;
use Illuminate\Http\JsonResponse;
use Modules\Auth\Actions\AuthenticateUser;
use Modules\Auth\Requests\LoginRequest;
use Modules\Auth\Resources\AccessTokenResource;

final class LoginController
{
    use ApiResponses;

    public function __construct(
        private readonly AuthenticateUser $authenticateUser,
    ) {
    }

    // Authenticating and issuing a token is a domain rule, not a one-liner
    // -> delegates to an Action.
    public function __invoke(LoginRequest $request): JsonResponse
    {
        $token = $this->authenticateUser->handle(
            email: $request->string('email')->toString(),
            password: $request->string('password')->toString(),
        );

        return $this->success(new AccessTokenResource($token));
    }
}
```

## Choosing between the two shapes

Matching a method name like `store` to the CRUD list is **not** the test.
The CRUD (resource) shape is only for a module that genuinely exposes
multiple resource operations on the same model (at least two of
`index`/`show`/`store`/`update`/`destroy` on that controller). A module
that only ever does one thing to a resource — for example, a `Comment`
module that supports creating a comment but has no `index`, `show`,
`update`, or `destroy` — is a single verb-shaped operation even though its
one method would be named `store`. That gets the invoke shape instead:
`final class StoreCommentController` with a single `__invoke()`, exactly
like `LoginController` above, not a one-method `CommentController` with a
`store()` method.

## 2. CRUD (resource) controller

For standard resource operations. `final class`, one method per operation
(`index`, `show`, `store`, `update`, `destroy`).

```php
<?php

declare(strict_types=1);

namespace Modules\User\Controllers;

use App\Http\Traits\ApiResponses;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Gate;
use Modules\User\Actions\CreateUser;
use Modules\User\Actions\DeleteUser;
use Modules\User\Actions\UpdateUser;
use Modules\User\Models\User;
use Modules\User\Requests\StoreUserRequest;
use Modules\User\Requests\UpdateUserRequest;
use Modules\User\Resources\UserResource;

final class UserController
{
    use ApiResponses;

    public function __construct(
        private readonly CreateUser $createUser,
        private readonly UpdateUser $updateUser,
        private readonly DeleteUser $deleteUser,
    ) {
    }

    // Plain read: a one-off, non-reusable paginated query -> no Action needed.
    public function index(): JsonResponse
    {
        return $this->success(UserResource::collection(User::query()->paginate()));
    }

    // Plain read: single-record lookup, no filtering or cross-model assembly.
    public function show(User $user): JsonResponse
    {
        Gate::authorize('view', $user);

        return $this->success(new UserResource($user));
    }

    // Mutates data -> delegates to an Action.
    public function store(StoreUserRequest $request): JsonResponse
    {
        Gate::authorize('create', User::class);

        $user = $this->createUser->handle($request->validated());

        return $this->success(new UserResource($user), 201);
    }

    // Mutates data -> delegates to an Action.
    public function update(UpdateUserRequest $request, User $user): JsonResponse
    {
        Gate::authorize('update', $user);

        $user = $this->updateUser->handle($user, $request->validated());

        return $this->success(new UserResource($user));
    }

    // Mutates data -> delegates to an Action.
    public function destroy(User $user): JsonResponse
    {
        Gate::authorize('delete', $user);

        $this->deleteUser->handle($user);

        return $this->success(null, 204);
    }
}
```

## The slim-controller rule

A controller method may contain a simple, one-off, non-reusable read
directly — `index`/`show` doing a plain Eloquent query, as above. But
**any** domain logic or data mutation must be delegated to an Action.

This isn't a CRUD-vs-not distinction; it's about reuse. The same Action
must be callable identically from an HTTP controller, an artisan command,
a queued job, or an MCP server, with the same inputs. If a "read" stops
being a one-liner — filtering, authorization-shaped assembly, cross-model
composition — it graduates to an Action too. See
`references/actions-and-services.md`.

Controllers own: request validation (via `Requests/`), authorization
(`Gate::authorize()`), calling the Action, and shaping the HTTP response
(via `Resources/` plus a shared response-formatting trait — see
`references/php-style.md`). Controllers do not own business logic.
