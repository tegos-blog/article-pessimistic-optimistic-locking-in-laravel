
_Managing Concurrent Data Safely_

Concurrency issues can be a headache in any system, particularly complex Laravel applications that could possibly deal
with sensitive information and financial data; double transactions, inconsistent reads or writing over some other models
are common traps laid down for the unassuming.
Here come the **pessimistic** and **optimistic** locking.
In this post, I will talk about both and present how you can use them in Laravel.

## What Is Locking?

Locking is a method dedicated at concurrency control which supports to avoid more than one course of action or consumer
from interfering with the database via getting access or rewriting identical data.

The two principal strategies are:

- **Pessimistic Locking** – "Don't let anyone touch it while I'm working"
- **Optimistic Locking** – "Assume it'll go ok. Check before saving."

## Pessimistic Locking in Laravel

Pessimistic locking works by *locking* the selected rows for the duration of a transaction. It ensures no other process
can modify (and sometimes even read) the data you're working with.

### Use Case

For instance, two users are trying to claim the same coupon code. Without locking, both can get
it. In this case, the first one locks it (pessimistic locking), and the second has to wait.

### `lockForUpdate()`

This locks the selected rows for **update**, and any other queries writing to this row would have to wait until your
transaction was complete.

```php
DB::transaction(function () {
    $user = DB::table('users')
        ->where('id', 1)
        ->lockForUpdate()
        ->first();

    // Safe to modify $user here
});
```

- Only within a transaction.
- Sessions perform other trying to `SELECT ... FOR UPDATE` on the same rows will wait (or timeout depending on DB
  settings).

### `sharedLock()`

This causes a **shared lock** to be taken out-other transactions could read but do any updates or deletes on this row.

```php
DB::transaction(function () {
    $user = DB::table('users')
        ->where('id', 1)
        ->sharedLock()
        ->first();
});
```

## Optimistic Locking in Laravel

Optimistic locking takes a different approach: it assumes that most operations won't conflict. It doesn’t lock rows at
the DB level but checks for changes before saving.

Laravel doesn’t offer optimistic locking out-of-the-box, but it’s simple to implement with a `version` or `updated_at`
column. Since updated_at is internal eloquent model column, it's better to use such like version column: version, track,
number.

### Use Case

Think of a product stock count in a high-traffic store. Instead of locking the row every time someone adds to cart, just
check if it’s been modified before updating.

### Custom Implementation (Using `updated_at`)

```php
$product = Product::query()->find($id);

$originalUpdatedAt = $product->updated_at;

// ... perform logic ...

$success = Product::where('id', $id)
    ->where('updated_at', $originalUpdatedAt)
    ->update([
        'stock' => $newStock,
    ]);

if (! $success) {
    // Another process updated it - retry or throw an error StaleModelLockingException
}
```

Or with a dedicated `version` column:

```php
$version = $product->version;

$success = Product::query()->where('id', $id)
    ->where('version', $version)
    ->update([
        'stock' => $newStock,
        'version' => $version + 1,
    ]);
```

## When to Use Which?

| Scenario                                        | Use                              |
|-------------------------------------------------|----------------------------------|
| High-conflict operations (e.g., bank transfers) | **Pessimistic**                  |
| Low-conflict but frequent reads/writes          | **Optimistic**                   |
| Long-running transactions                       | **Optimistic** (locks are risky) |
| You need guaranteed update order                | **Pessimistic**                  |

## Gotchas to Watch Out For

- **Deadlocks**: Pessimistic locks can cause deadlocks if not carefully ordered.
- **Timeouts**: Long-running transactions can block others.
- **Retries**: Optimistic locking requires retry logic on failure.

## Practical Implementation Example

Here's a real-world example from a system that processes order cancellations using pessimistic locking:

```php
final class OrderItemCancelAction implements Actionable
{
    public function __construct(private readonly AdminRepository $adminRepository)
    {
    }

    public function handle(string $orderItemUuId, int $userId, ?int $managerId = null): void
    {
        $admin = $this->adminRepository->getAdmin($managerId);

        $orderItem = OrderItem::query()->where('is_canceled', false)->findOrFail($orderItemUuId);

        DB::transaction(function () use ($userId, $orderItem, $admin) {
            /** @var Order $order */
            $order = Order::query()
                ->lockForUpdate()
                ->where('user_id', $userId)
                ->whereIn('status_id', [
                    OrderStatusEnum::PROCESSING->value,
                    OrderStatusEnum::SUSPENDED->value,
                ])
                ->findOrFail($orderItem->order_id);

            $orderItem->update([
                'is_canceled' => true,
                'canceled_at' => Carbon::now(),
                'canceled_by' => $admin->id ?? 0,
                'status_id' => OrderItemStatusEnum::REFUSAL_CLIENT->value,
            ]);

            $hasNonCanceledItems = $order->items()->where('is_canceled', false)->exists();

            $attributes = [
                'price' => $order->items()->where('is_canceled', false)->sum(DB::raw('price * quantity')),
            ];

            if (!$hasNonCanceledItems) {
                $attributes['status_id'] = OrderStatusEnum::CANCELLED->value;
            }

            $order->update($attributes);
        });
    }
}
```

## Implementation Best Practices

1. **Lock Order**: Always acquire locks in the same order to avoid deadlocks
2. **Minimize Lock Duration**: Keep transactions as short as possible
3. **Lock Granularity**: Lock only the records you need to modify
4. **Choose Wisely**: Use `sharedLock()` when you only need to read consistently
5. **Proper Execution Order**: Remember that `lockForUpdate()` and `sharedLock()` must be called before the query
   executes:
   ```php
   // CORRECT: Lock applied before query execution
   $model = User::lockForUpdate()->find(1);
   
   // INCORRECT: Lock applied after query has already executed
   $model = User::find(1)->lockForUpdate();
   ```

## Conclusion

Laravel's `lockForUpdate()` and `sharedLock()` provide powerful tools for preventing race conditions in your database
transactions. By understanding how they work and when to use each, you can ensure your application maintains data
integrity even under high concurrency situations.

Remember:

- Use database transactions to ensure atomicity
- Use `lockForUpdate()` when you need to read and then update records
- Use `sharedLock()` when you need consistent reads without modification
- Consider optimistic locking for better performance in low-contention scenarios

Implementing proper locking strategies will help your Laravel applications handle concurrent operations reliably,
preventing data inconsistencies and unexpected behavior.