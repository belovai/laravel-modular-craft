# PHP style

General class conventions across the module system — not just
Actions/Services/VOs/DTOs:

- `declare(strict_types=1);` at the top of every PHP file.
- `final class` by default; only remove `final` when inheritance is a
  deliberate, documented design choice.
- `readonly` properties wherever the value doesn't need to change after
  construction (common on VOs, DTOs, Services).
- Constructor property promotion instead of manually declared properties
  plus assignment.
- Enums instead of magic strings/constants for closed sets of values,
  including custom helper methods on the enum when useful:

```php
<?php

declare(strict_types=1);

namespace Modules\User\Enums;

enum UserStatus: string
{
    case Active = 'active';
    case Suspended = 'suspended';
    case Deleted = 'deleted';

    public function notEquals(self $other): bool
    {
        return $this !== $other;
    }
}
```

- API response formatting is extracted into a shared trait — the
  convention, not a specific trait name or implementation. (The reference
  examples in this skill use `ApiResponses`, but that trait itself is
  app-level/cross-module, not part of the module-authoring pattern this
  skill teaches.)
