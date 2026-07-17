# Actions and Services

## Action — the default home for domain logic and data mutation

- One class, one job: named `VerbNoun` (`CreateUser`, `PurgeTokens`,
  `GenerateToken`). `final class`, single public `handle()` method.
- Exists specifically so the same operation is reusable, with identical
  inputs/output, from multiple entry points: HTTP controller, artisan
  command, queued job, MCP server tool.
- If logic will only ever be called from one controller method and
  involves no real domain rule, a plain controller one-liner is fine (see
  `references/controllers.md`) — but default to an Action once there's
  any real logic.
- Actions can depend on Services and other Actions via constructor
  injection.

```php
<?php

declare(strict_types=1);

namespace Modules\User\Actions;

use Illuminate\Support\Facades\Hash;
use Modules\User\Models\User;

final class CreateUser
{
    /**
     * @param array{name: string, email: string, password: string} $data
     */
    public function handle(array $data): User
    {
        return User::query()->create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
        ]);
    }
}
```

## Service — for logic shared beyond a single Action, or wrapping an external system

- `final readonly class`, constructor-promoted dependencies (e.g. base
  URL, credentials, an injected HTTP client).
- When a Service wraps an external system, define a `Contracts/` interface
  for it and bind the concrete implementation in the module's provider —
  this keeps the Action layer testable/mockable against the interface.
- Do not use a Service just to avoid a slightly longer Action — only
  reach for one when logic is genuinely shared across Actions or is
  externally-vendored.

```php
<?php

declare(strict_types=1);

namespace Modules\Probe\Contracts;

interface QueueManagementService
{
    /**
     * @return array<int, array{name: string, messages: int}>
     */
    public function listQueues(string $vhost): array;

    public function purgeQueue(string $vhost, string $queue): void;
}
```

```php
<?php

declare(strict_types=1);

namespace Modules\Probe\Services;

use GuzzleHttp\ClientInterface;
use Modules\Probe\Contracts\QueueManagementService;

final readonly class RabbitMqManagementService implements QueueManagementService
{
    public function __construct(
        private ClientInterface $http,
        private string $baseUrl,
        private string $username,
        private string $password,
    ) {
    }

    public function listQueues(string $vhost): array
    {
        $response = $this->http->request('GET', "{$this->baseUrl}/api/queues/{$vhost}", [
            'auth' => [$this->username, $this->password],
        ]);

        return json_decode($response->getBody()->getContents(), true);
    }

    public function purgeQueue(string $vhost, string $queue): void
    {
        $this->http->request('DELETE', "{$this->baseUrl}/api/queues/{$vhost}/{$queue}/contents", [
            'auth' => [$this->username, $this->password],
        ]);
    }
}
```

Bind the contract to the implementation in the module's provider (see
`references/architecture.md`):

```
$this->app->bind(QueueManagementService::class, RabbitMqManagementService::class);
```
