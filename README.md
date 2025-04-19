# Pessimistic & Optimistic Locking in Laravel

![Pessimistic & Optimistic Locking in Laravel](assets/poster.jpg)

**Safeguard Your Laravel App from Race Conditions and Data Conflicts**

Concurrency issues - like double transactions, lost updates, and inconsistent reads—can quietly break your application
logic and data integrity. This article breaks down the two primary approaches to handling concurrent access to data:
**pessimistic locking** and **optimistic locking**, and how to implement them effectively in Laravel.

## 🔒 Pessimistic Locking with Laravel's Query Builder

Pessimistic locking uses database-level row locks to prevent other operations from modifying or even reading data during
a transaction. Learn how to use `lockForUpdate()` and `sharedLock()` in Laravel's query builder to secure critical
operations like order processing, coupon redemption, or stock allocation.

## 🔄 Optimistic Locking with Versioning

Optimistic locking assumes that conflicts are rare. Instead of locking rows, it checks whether the data was changed
before committing updates. While Laravel doesn't support it out of the box, this post shows you how to implement it
using a `version` or `updated_at` column—ideal for high-read, low-conflict scenarios.

## Best Practices and Real-world Example

You'll also find a practical use case from an order cancellation system using `lockForUpdate()` safely inside
transactions. Plus, a checklist of best practices to avoid common pitfalls like deadlocks, timeouts, and inconsistent
reads.

## Learn More

Read the full article: [Pessimistic & Optimistic Locking in Laravel](https://dev.to/tegos/pessimistic-optimistic-locking-in-laravel-23dk)

