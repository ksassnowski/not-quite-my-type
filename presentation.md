build-lists: true
footer: Not Quite My Type: Using types to make impossible states truly impossible

# Not Quite My Type
### _Using types to make impossible states truly impossible_

[.footer: Laravel Meetup Munich June 2022]

---

# Scenario 1: **Device IDs**

---

## Scenario 1: Device IDs

^Here’s what we know about device ids

- Every **device** has an **ID**
- Devices are associated with users
- A user has either **zero** or **exactly one** device
- Devices are **not stored locally**
- Device IDs are **UUIDS**

---

[.code-highlight: all]
[.code-highlight: 2]

```php
/**
 * @property ?string $device_id
 */
final class User extends Model
{
}
```

---

[.code-highlight: all]
[.code-highlight: 5-7]
[.code-highlight: 9-11]
[.code-highlight: 13]

```php
final class AssociateDeviceWithUserController
{
    public function __invoke(Request $request, User $user): RedirectResponse
    {
        $validated = $request->validate([
            'device_id' => ['required', 'string', 'uuid'],
        ]);

        $user->update([
            'device_id' => $validated['device_id'],
        ]);

        return redirect()->back();
    }
}
```

---

[.code-highlight: 9]

```php
final class AssociateDeviceWithUserController
{
    public function __invoke(Request $request, User $user): RedirectResponse
    {
        $validated = $request->validate([
            'device_id' => ['required', 'string', 'uuid'],
        ]);

        $user->assignDevice($validated['device_id']);

        return redirect()->back();
    }
}
```

---

[.code-highlight: all]
[.code-highlight: 6-9]

```php
/**
 * @property ?string $device_id
 */
final class User extends Model
{
    public function assignDevice(string $deviceID): void
    {
        $this->update(['device_id' => $deviceID]);
    }
}
```

---

## We got a problem

---

```php
// WRONG: Device ID cannot be empty
$user->assignDevice('');
```

---

```php
// WRONG: Device ID cannot be empty
$user->assignDevice('')

// WRONG: Device ID is not a valid UUID
$user->assignDevice('not a uuid lol')
```

---

```php
// WRONG: Device ID cannot be empty
$user->assignDevice('')

// WRONG: Device ID is not a valid UUID
$user->assignDevice('not a uuid lol')
```

## Both calls are **syntactically correct**!

---

## But we’ve validated the request payload!

---

## Funny you say that...

---

## Now also available in **command form**

[.code-highlight: all]
[.code-highlight: 7]
[.code-highlight: 8-10]

```php
final class AssignDeviceCommand extends Command
{
    protected $signature = 'app:assign-device {userID} {deviceID}';

    public function handle(): int
    {
        $user = User::findOrFail($this->argument('userID'));
        $deviceID = $this->argument('deviceID');

        $user->assignDevice($deviceID);

        return self::SUCCESS;
    }
}
```

---

## Uh oh

```bash
# WRONG: Device ID cannot be empty
php artisan app:assign-device 1 ''
```

---

## Uh oh

```bash
# WRONG: Device ID cannot be empty
php artisan app:assign-device 1 ''
```

```bash
# WRONG: Device ID needs to be a UUID
php artisan app:assign-device 1 'not a uuid lol'
```

---

> We’re assuming that the device ID has been validated **by the caller**

---

```php
public function assignDevice(string $deviceID): void
{
    $this->update(['device_id' => $deviceID]);
}
```

<br>

This implementation is only correct if `$deviceID` was validated **before** the method got called.

Otherwise, we’re saving **invalid state**.

---

## There is an **implicit dependency** between the method and its caller

---

## Alright, I’ll add the **if** already...

```diff
public function assignDevice(string $deviceID): void
{
+   if (!Uuid::isValid($deviceID)) {
+       throw new InvalidArgumentException(
+           'Device ID needs to be a valid UUID',
+       );
+   }

    $this->update(['device_id' => $deviceID]);
}
```

---

# Crisis **averted** :+1:

---

# Looking up devices **by ID**

---

```php
final class DeviceRepository 
    implements DeviceRepositoryInterface
{
    public function find(string $deviceID): ?Device
    {
        // Call external API to find a device by its ID.
    }
}
```

---

```php
$repository = new DeviceRepository();

$user = User::firstOrFail();

// Let's assume the user's device_id isn't null
$deviceOrNull = $repository->find($user->device_id);
```

---

```php
$repository = new DeviceRepository();

$user = User::firstOrFail();

// Let's assume the user's device_id isn't null
$deviceOrNull = $repository->find($user->device_id);
```

:+1:

---

# Don’t you **dare**...

---

## Whoops

```php
$repository = new DeviceRepository();

$guaranteedNotADevice = $repository->find('not a uuid, lol');
```

---

## Whoops

```php
$repository = new DeviceRepository();

$guaranteedNotADevice = $repository->find('not a uuid, lol');
```

<br>

This is a perfectly valid use of the `find` method. It’s also **complete nonsense**.

---

## sigh...


```diff
final class DeviceRepository 
    implements DeviceRepositoryInterface
{
    public function find(string $deviceID): ?Device
    {
+       if (!Uuid::isValid($deviceID)) {
+           return null;
+       }

        // Call external API to find a device by its ID.
    }
}
```

---

# Crisis **averted** :+1:

---

# Wait, what's that **smell?**

---

[.code-highlight: all]
[.code-highlight: 5-9, 20-22]

[.column]

```php
final class User extends Model
{
    public function assignDevice(string $deviceID): void
    {
        if (!Uuid::isValid($deviceID)) {
            throw new InvalidArgumentException(
                'Device ID needs to be a valid UUID',
            );
        }

        $this->update(['device_id' => $deviceID]);
    }
}

final class DeviceRepository 
    implements DeviceRepositoryInterface
{
    public function find(string $deviceID): ?Device
    {
        if (!Uuid::isValid($deviceID)) {
            return null;
        }

        // Call external API to find a device by its ID.
    }
}
```

[.column]

### 🤔

---

[.column]

[.code-highlight: 5-9, 20-22]

```php
final class User extends Model
{
    public function assignDevice(string $deviceID): void
    {
        if (!Uuid::isValid($deviceID)) {
            throw new InvalidArgumentException(
                'Device ID needs to be a valid UUID',
            );
        }

        $this->update(['device_id' => $deviceID]);
    }
}

final class DeviceRepository 
    implements DeviceRepositoryInterface
{
    public function find(string $deviceID): ?Device
    {
        if (!Uuid::isValid($deviceID)) {
            return null;
        }

        // Call external API to find a device by its ID.
    }
}
```

[.column]

- It’s possible to call **assignDevice** and **find** with nonsense values
- We have to worry about the **invariants** of device IDs **every time we deal with them**

---

# A device ID isn’t a `string`

---

# `strings` can be empty, a device ID can’t

---

# `strings` don’t have to be valid UUIDs, a device ID does

---

# A device ID is a `UUID`

---

# Let’s **narrow** our types

--- 

```diff
final class DeviceRepository 
    implements DeviceRepositoryInterface
{
-   public function find(string $deviceID): ?Device
+   public function find(UuidInterface $deviceID): ?Device
    {
-       if (!Uuid::isValid($deviceID)) {
-           return null;
-       }
-
        // Call external API to find a device by its ID.
    }
}
```

---

```diff
final class User extends Model
{
-   public function assignDevice(string $deviceID): void
+   public function assignDevice(UuidInterface $deviceID): void
    {
-       if (!Uuid::isValid($deviceID)) {
-           throw new InvalidArgumentException(
-               'Device ID needs to be a valid UUID',
-           );
-       }
- 
        $this->update(['device_id' => $deviceID]);
    }
}
```

---

# Let’s try to **break it**

---

```php
$user = User::firstOrFail();

// TypeError: expected UuidInterface, got string
$user->assignDevice('not a uuid, lol');
```

---

```php
$user = User::firstOrFail();

// TypeError: expected UuidInterface, got string
$user->assignDevice('not a uuid, lol');

// Throws an exception, because it's not a valid UUID
$uuid = Uuid::fromString('not a uuid, lol');
```

---

```php
$user = User::firstOrFail();

// TypeError: expected UuidInterface, got string
$user->assignDevice('not a uuid, lol');

// Throws an exception, because it's not a valid UUID
$uuid = Uuid::fromString('not a uuid, lol');
```

<br>

It’s **impossible** to even call the method with an **invalid parameter**.

---

# There’s still a **problem**, however...

---

This still returns `?string`

```php
$deviceID = $user->device_id;
```

---

This still returns `?string`

```php
$deviceID = $user->device_id;
```

So we cannot do this anymore

```php
$repository = new DeviceRepository();

// TypeError: expected UuidInterface, got string
$repository->find($user->device_id);
```

---

# Using **Attribute Casts**
#### link to docs here

---

[.code-highlight: all]
[.code-highlight: 3-6]
[.code-highlight: 5]
[.code-highlight: 8-17]
[.code-highlight: 10-13]
[.code-highlight: 15]

```php
final class UUIDCast implements CastsAttributes
{
    public function get($model, $key, $value, $attributes)
    {
        return $value !== null ? Uuid::fromString($value) : null;
    }

    public function set($model, $key, $value, $attributes)
    {
        if ($value !== null && !$value instanceof UuidInterface) {
            throw new InvalidArgumentException(
                'Provided value is not an instance of UuidInterface',
            );
        }

        return [$key => $value->toString()];
    }
}
```

---

[.code-highlight: all]
[.code-highlight: 6-8]
[.code-highlight: 2]

```php
/**
 * @property ?UuidInterface $device_id
 */
final class User extends Model
{
    protected $casts = [
        'device_id' => UUIDCast::class,
    ];

    /* snip */
}
```

---

Using this cast, `device_id` can now only be set to `null`, or a `UuidInterface`.

```php
// works, null is allowed
$user->device_id = null;

// works, is a valid UUID
$user->device_id = Uuid::uuid4();
```

---

`device_id` can now **only** be set to `null`, or a `UuidInterface`.

```php
// works, null is allowed
$user->device_id = null;

// works, is a valid UUID
$user->device_id = Uuid::uuid4();

// InvalidArgumentException: Provided value is not 
// an instance of UuidInterface
$user->device_id = 'not a uuid, lol';
```

---

`device_id` is **guaranteed** to be either `null` or a valid `UuidInterface`

```php
$deviceID = $user->device_id;
```

---

`device_id` is **guaranteed** to be either `null` or a valid `UuidInterface`

```php
$deviceID = $user->device_id;
```

Meaning we can now do this again

```php
if ($deviceID === null) {
    return;
}

// type checks since $deviceID is null|UuidInterface 
// and we've checked for null above
$repository->find($deviceID);
```

---

# Can we do **even better**?

---

[.code-highlight: all]
[.code-highlight: 2, 8]
[.code-highlight: 3]

```php
/**
 * @property UuidInterface $uuid
 * @property UuidInterface $device_id
 */
final class User extends Model
{
    protected $casts = [
        'uuid' => UUIDCast::class,
        'device_id' => UUIDCast::class,
    ];
}
```

---

```php
$deviceRepository = new DeviceRepository();

$user = User::firstOrFail();

// ❌ This type-checks but is clearly nonsense
$definitelyNotADevice = $deviceRepository->find($user->uuid);
```

---

# A device ID is not a `UuidInterface`

--- 

## If it was, this call should **make sense**

```php
$deviceRepository->find($user->uuid);
```

---

## If it was, this call should **make sense**

```php
$deviceRepository->find($user->uuid);
```

## But we know it’s **nonsense**

---

## So let’s make it nonsense <br> **at the type level**

---

# A device ID is a `DeviceID`

---

[.code-highlight: all]
[.code-highlight: 3-6]
[.code-highlight: 8-17]
[.code-highlight: 10-14]
[.code-highlight: 16]
[.code-highlight: all]

```php
final class DeviceID
{
    private function __construct(
        public readonly string $id,
    ) {
    }

    public static function fromString(string $id): self
    {
        if (!Uuid::isValid($id)) {
            throw new InvalidArgumentException(
                'A device ID needs to be be valid UUID'
            );
        }

        return new self($id);
    }
}
```

---

[.code-highlight: all]
[.code-highlight: 2, 9-11]

```php
/**
 * @property DeviceID      $device_id
 * @property UuidInterface $uuid
 */
final class User extends Model
{
    protected $casts = [
        'uuid' => UUIDCaster::class,
        // Implementation of DeviceIDCaster is left as
        // an excercise to the listener.
        'device_id' => DeviceIDCaster::class,
    ];
}
```

---

```php
// ✅ TypeError: Expected DeviceID but got UuidInterface
$deviceRepository->find($user->uuid);
```

---

```php
// ✅ TypeError: Expected DeviceID but got UuidInterface
$deviceRepository->find($user->uuid);


// ✅ Works
$deviceRepository->find($user->device_id);
```

---

# Crisis **averted** :+1:
##### For real this time

---

# Scenario 2: **Price Discount**

---

```php
final class ShoppingCart
{
    public function __construct(private array $products)
    {
    }

    public function total(): int
    {
        $total = 0;

        foreach ($this->products as $product) {
            $total += $product->price * product->quantity;
        }

        return $total;
    }
}
```

---

# Applying a discount

---

[.code-highlight: all]
[.code-highlight: 3]
[.code-highlight: 9-12]
[.code-highlight: 22-24]

```php
final class ShoppingCart
{
    private float $discount = 0.0;

    public function __construct(private array $products)
    {
    }

    public function applyDiscount(float $discount): void
    {
        $this->discount = $discount;
    }

    public function total(): int
    {
        $originalPrice = 0;

        foreach ($this->products as $product) {
            $originalPrice += $product->price * product->quantity;
        }

        $salePrice = $originalPrice * (1.0 - $this->discount);

        return (int) round($salePrice, 0);
    }
}
```

---

# 🤔

---

```php
// Pointless
$cart->applyDiscount(0.0);

// Apply a very seller-friendly "discount"
$cart->applyDiscount(-0.50);
```

---

```php
public function applyDiscount(float $discount): void
{
    if ($discount <= 0.0) {
        throw new InvalidArgumentException(
            'Discount has to be a positive float'
        );
    }

    $this->discount = $discount;
}
```

---

# A discount is not a `float`

---

# `floats` can be negative, a discount can’t

---

# `floats` can be zero, a discount can’t

---

# Let’s narrow our types

---

[.code-highlight: all]
[.code-highlight: 3-6]
[.code-highlight: 8]
[.code-highlight: 10-14]
[.code-highlight: 16]


```php
final class Discount
{
    private function __construct(
        public readonly float $percentage,
    ) {
    }

    public static function percentOff(int $value): self
    {
        if ($value <= 0) {
            throw new InvalidArgumentException(
                'Discount needs to be a positive, non-zero value',
            );
        }

        return new self($value / 100);
    }
}
```

---

```diff
final class ShoppingCart
{
-   private float $discount = 0.0
+   private ?Discount $discount = null;

    public function __construct(private array $products)
    {
    }

-    public function applyDiscount(float $discount): void
+    public function applyDiscount(Discount $discount): void
    { 
-       if ($discount <= 0.0) {
-           throw new InvalidArgumentException(
-               'Discount has to be a positive float'
-           );
-       }
-
        $this->discount = $discount;
    }
```

---

```diff
    public function total(): int
    {
        $originalPrice = 0;

        foreach ($this->products as $product) {
            $originalPrice += $product->price * product->quantity;
        }

+       if ($this->discount === null) {
+           return $originalPrice;
+       }
+
-       $salePrice = $originalPrice * (1.0 - $this->discount);
+       $salePrice = $originalPrice * (1.0 - $this->discount->percentage);

        return (int) round($salePrice, 0);
    }
}
```

---

# Let’s improve our API

---

## 1. Make Discount opaque

```diff
final class Discount
{
    private function __construct(
-       public readonly float $percentage,
+       private readonly float $percentage,
    ) {
    }
```

---

## 2. Apply discount

```diff
final class Discount
{
    private function __construct(
        private readonly float $percentage,
    ) {
    }

+   public function apply(int $originalPrice): int
+   {
+       $discountedPrice = $originalPrice * (1.0 - $this->percentage);
+
+       return (int) round($discountedPrice, 0);
+   }
}
```

---

## 3. Refactor `ShoppingCart`

```diff
final class ShoppingCart
{
    public function total(): int
    {
        $originalPrice = 0;

        foreach ($this->products as $product) {
            $originalPrice += $product->price * product->quantity;
        }

        if ($this->discount === null) {
            return $originalPrice;
        }

-       $salePrice = $originalPrice * (1.0 - $this->discount);
-
-       return (int) round($salePrice, 0);
+       return $this->discount->apply($originalPrice);
    }
}
```

---

## 4. Putting it all together

```php
$discount = Discount::percentOff(15);

$cart->applyDiscount($discount);

$cart->total();

😍
```

---

- If an **int** can't be negative, it’s not an **int** because an **int** can be negative
- If a **string** can't be empty, it’s not a **string** because a **string** can be empty
- If a **string** has to have a specific length, it’s not a **string** because a **string** can have any length

---

> If a parameter has to satisfy certain invariants, consider promoting it to an object that **guarantees these invariants**