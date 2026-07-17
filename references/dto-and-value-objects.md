# DTOs and Value Objects

Both live under module subfolders spelled out in full:
`DataTransferObjects/` and `ValueObjects/` — never `Dto`/`VO` abbreviated
in the namespace or folder.

**The core rule driving both: never return an array just because a method
needs to return 2+ related values.** If it's a single meaningful value →
VO. If it's a larger, more request/response-shaped bag of related fields
→ DTO.

## Value Object (VO)

A small, cohesive, self-meaningful value — `final readonly class`,
constructor property promotion only, public readonly properties, no
`fromArray`/`toArray` machinery. Use when the value is meaningful standing
alone (e.g. `AccessToken`, `PlainToken`, `Money`).

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

## DTO

For carrying a larger bag of related data across a boundary instead of
returning/passing a bare array. Can have `fromArray()`/`toArray()` (or
implement Laravel's `Arrayable`) when the data genuinely needs to move in
and out of array form — e.g. a request payload becoming a DTO becoming an
external API call body:

```php
<?php

declare(strict_types=1);

namespace Modules\Invoice\DataTransferObjects;

final readonly class CustomerDTO
{
    public function __construct(
        public string $name,
        public string $taxNumber,
        public string $email,
        public ?string $billingAddress,
    ) {
    }

    /**
     * @param array{name: string, tax_number: string, email: string, billing_address?: string|null} $data
     */
    public static function fromArray(array $data): self
    {
        return new self(
            name: $data['name'],
            taxNumber: $data['tax_number'],
            email: $data['email'],
            billingAddress: $data['billing_address'] ?? null,
        );
    }

    /**
     * @return array{name: string, tax_number: string, email: string, billing_address: string|null}
     */
    public function toArray(): array
    {
        return [
            'name' => $this->name,
            'tax_number' => $this->taxNumber,
            'email' => $this->email,
            'billing_address' => $this->billingAddress,
        ];
    }
}
```

Do **not** bake JSON serialization concerns into every DTO by default. If
a DTO needs `JsonSerializable` or similar, that behavior should come from
a shared trait (e.g. a `Serializable`-style trait), not be hand-rolled per
DTO. This is a nice-to-have, not a required part of the DTO convention —
don't add it speculatively.
