# Testing

- Tests live **inside the module**: `modules/{Module}/Tests/Feature/` and
  `modules/{Module}/Tests/Unit/`. Never centralize module tests under a
  top-level `tests/` directory.
- Pest syntax (`describe`/`it`), not PHPUnit class-based tests.
- Feature tests exercise the HTTP surface end-to-end (route → controller
  → action → response), naming the file after the operation
  (`CreateUserTest`, `DestroyUserTest`).
- Unit tests target a single Action/Service/VO/DTO in isolation when the
  behavior warrants it — not required for every trivial class.

`modules/User/Tests/Feature/CreateUserTest.php`:

```php
<?php

declare(strict_types=1);

use Modules\User\Models\User;

describe('POST /api/users', function () {
    it('creates a user with valid data', function () {
        $response = $this->postJson('/api/users', [
            'name' => 'Ada Lovelace',
            'email' => 'ada@example.com',
            'password' => 'correct-horse-battery-staple',
        ]);

        $response->assertCreated();

        $this->assertDatabaseHas('users', [
            'email' => 'ada@example.com',
        ]);
    });

    it('rejects a duplicate email', function () {
        User::factory()->create(['email' => 'ada@example.com']);

        $response = $this->postJson('/api/users', [
            'name' => 'Ada Lovelace',
            'email' => 'ada@example.com',
            'password' => 'correct-horse-battery-staple',
        ]);

        $response->assertUnprocessable();
    });
});
```
