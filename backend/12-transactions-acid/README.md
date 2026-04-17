# Database Transactions and ACID Properties

Your payment succeeded, but the order wasn't created. Money moved, inventory deducted—but the order table insert failed halfway through. Now you've got angry customers, missing inventory, and a support ticket nightmare.

This is what happens when you treat database operations as independent events. Transactions are the safety net that keeps your data consistent when things go wrong—and things *will* go wrong.

## What's a Transaction?

A transaction is a group of database operations that succeed or fail together. It's all-or-nothing:

```sql
BEGIN;

-- Deduct from user's balance
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;

-- Add to merchant's balance
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- Create the order
INSERT INTO orders (user_id, amount, status) VALUES (1, 100, 'completed');

COMMIT;
```

If any statement fails, the entire transaction rolls back. No partial state. No missing money.

## ACID: The Four Guarantees

### Atomicity: All or Nothing

Atomicity means the transaction is indivisible. Either all operations complete, or none do.

```
❌ Without atomicity:
1. Deduct $100 from user ✓
2. Add $100 to merchant ✗ (constraint violation)
   → User lost $100, merchant got nothing, order never created

✅ With atomicity:
1. Deduct $100 from user ✓
2. Add $100 to merchant ✗ (constraint violation)
   → Entire transaction rolls back
   → User balance restored, no chaos
```

### Consistency: From Valid State to Valid State

Consistency ensures transactions take the database from one valid state to another. Constraints, triggers, and cascades all enforce this.

```sql
-- ❌ Violates consistency (balance goes negative)
UPDATE accounts SET balance = balance - 500 WHERE user_id = 1;
-- Rejected if CHECK constraint: balance >= 0

-- ✅ Maintains consistency
UPDATE accounts SET balance = balance - 50 WHERE user_id = 1;
-- Satisfies all constraints
```

### Isolation: Transactions Don't Interfere

Isolation means concurrent transactions don't affect each other. What happens inside a transaction stays invisible until commit.

```sql
-- Transaction A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
-- (not committed yet)

-- Transaction B (running at the same time)
SELECT balance FROM accounts WHERE user_id = 1;
-- With proper isolation, B sees the OLD balance
-- Not the intermediate state of A's uncommitted transaction
```

### Durability: Committed Means Committed

Once a transaction commits, it's permanent. Power failure, crash, restart—the data survives.

```sql
BEGIN;
INSERT INTO orders (user_id, amount) VALUES (1, 100);
COMMIT;
-- At this point, the order is guaranteed to exist
-- Even if the database crashes 1ms later
```

Databases achieve this through write-ahead logs (WAL). Changes are written to a log before being applied to the actual data.

## Isolation Levels: The Trade-offs

Isolation isn't binary—it's a spectrum. Higher isolation means more safety but less concurrency.

| Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Performance |
|-------|-------------|---------------------|---------------|-------------|
| Read Uncommitted | ❌ Possible | ❌ Possible | ❌ Possible | Fastest |
| Read Committed | ✅ Prevented | ❌ Possible | ❌ Possible | Fast |
| Repeatable Read | ✅ Prevented | ✅ Prevented | ❌ Possible | Moderate |
| Serializable | ✅ Prevented | ✅ Prevented | ✅ Prevented | Slowest |

### Dirty Read: Seeing Uncommitted Data

```sql
-- Transaction A
BEGIN;
UPDATE accounts SET balance = 500 WHERE user_id = 1;
-- Not committed yet!

-- Transaction B (Read Uncommitted)
SELECT balance FROM accounts WHERE user_id = 1;
-- Sees 500, but A might roll back!

-- Transaction A
ROLLBACK;
-- B made decisions based on data that never existed
```

### Non-Repeatable Read: Data Changes Mid-Transaction

```sql
-- Transaction A
SELECT balance FROM accounts WHERE user_id = 1;  -- Returns 100

-- Transaction B
UPDATE accounts SET balance = 200 WHERE user_id = 1;
COMMIT;

-- Transaction A
SELECT balance FROM accounts WHERE user_id = 1;  -- Returns 200!
-- Same query, different result within one transaction
```

### Phantom Read: New Rows Appear

```sql
-- Transaction A
SELECT COUNT(*) FROM orders WHERE status = 'pending';  -- Returns 5

-- Transaction B
INSERT INTO orders (status) VALUES ('pending');
COMMIT;

-- Transaction A
SELECT COUNT(*) FROM orders WHERE status = 'pending';  -- Returns 6!
-- A "phantom" row appeared
```

## Choosing an Isolation Level

### Read Committed (Default in PostgreSQL, SQL Server)

Good for most web applications:

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

- You see only committed data
- Each statement sees a snapshot from when it started
- Good balance of safety and performance

### Repeatable Read (Default in MySQL)

Stronger guarantees for read-heavy transactions:

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

- All reads within a transaction see the same snapshot
- Prevents non-repeatable reads
- MySQL also prevents phantom reads at this level

### Serializable: When Correctness is Non-Negotiable

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

Use for:
- Financial operations
- Inventory systems where overselling is unacceptable
- Any scenario where anomalies cause real damage

Trade-off: Performance. Serializable transactions may fail and need retry:

```javascript
// ❌ Write without retry
await db.transaction(async (tx) => {
  await tx.execute('UPDATE accounts SET balance = balance - 100');
});

// ✅ Retry on serialization failure
async function safeTransaction(fn, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await db.transaction(fn);
    } catch (err) {
      if (err.code === '40001' && i < retries - 1) continue; // Serialization failure
      throw err;
    }
  }
}
```

## Common Transaction Patterns

### 1. The Money Transfer

```sql
BEGIN;

-- Lock rows in consistent order to prevent deadlocks
SELECT balance FROM accounts WHERE user_id = 1 FOR UPDATE;
SELECT balance FROM accounts WHERE user_id = 2 FOR UPDATE;

-- Check sufficient balance
-- (application logic or stored procedure)

UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

INSERT INTO transfers (from_user, to_user, amount) VALUES (1, 2, 100);

COMMIT;
```

### 2. The Order with Inventory

```javascript
// ❌ Race condition waiting to happen
async function createOrder(userId, productId, quantity) {
  const product = await db.query('SELECT stock FROM products WHERE id = $1', [productId]);
  if (product.stock < quantity) throw new Error('Out of stock');
  
  await db.query('UPDATE products SET stock = stock - $1 WHERE id = $2', [quantity, productId]);
  await db.query('INSERT INTO orders (user_id, product_id, quantity) VALUES ($1, $2, $3)', [userId, productId, quantity]);
}

// ✅ Atomic with constraint check
async function createOrder(userId, productId, quantity) {
  await db.transaction(async (tx) => {
    // Lock the product row
    await tx.query('SELECT stock FROM products WHERE id = $1 FOR UPDATE', [productId]);
    
    // Update with condition (fails if stock goes negative)
    const result = await tx.query(
      'UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1',
      [quantity, productId]
    );
    
    if (result.rowCount === 0) throw new Error('Out of stock');
    
    await tx.query('INSERT INTO orders (user_id, product_id, quantity) VALUES ($1, $2, $3)', [userId, productId, quantity]);
  });
}
```

### 3. Batch Operations

```sql
-- ❌ Individual statements (slow, inconsistent state possible)
INSERT INTO logs (message) VALUES ('msg1');
INSERT INTO logs (message) VALUES ('msg2');
INSERT INTO logs (message) VALUES ('msg3');
-- What if msg2 fails? msg1 is already inserted.

-- ✅ Single transaction
BEGIN;
INSERT INTO logs (message) VALUES ('msg1');
INSERT INTO logs (message) VALUES ('msg2');
INSERT INTO logs (message) VALUES ('msg3');
COMMIT;
```

## Deadlocks: When Transactions Wait Forever

Two transactions, each waiting for the other to release a lock:

```
Transaction A: Locks row 1, wants row 2
Transaction B: Locks row 2, wants row 1
→ Both wait forever → Deadlock
```

### Prevention Strategies

```sql
-- ❌ Inconsistent lock order causes deadlock
-- Transaction A
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- Transaction B (running simultaneously)
UPDATE accounts SET balance = balance - 50 WHERE user_id = 2;
UPDATE accounts SET balance = balance + 50 WHERE user_id = 1;

-- ✅ Always lock in consistent order (e.g., by ID)
-- Both transactions now lock user 1 first, then user 2
```

```javascript
// Application-level deadlock handling
async function transfer(fromId, toId, amount) {
  // Sort IDs to ensure consistent lock order
  const [first, second] = [fromId, toId].sort((a, b) => a - b);
  
  return db.transaction(async (tx) => {
    await tx.query('SELECT * FROM accounts WHERE user_id = $1 FOR UPDATE', [first]);
    await tx.query('SELECT * FROM accounts WHERE user_id = $1 FOR UPDATE', [second]);
    // ... rest of transfer logic
  });
}
```

## Transaction Anti-Patterns

### 1. Long-Running Transactions

```javascript
// ❌ Transaction open during slow operation
await db.transaction(async (tx) => {
  await tx.query('UPDATE accounts SET balance = balance - 100');
  await fetch('https://payment-provider.com/charge'); // HTTP call inside transaction!
  await tx.query('UPDATE accounts SET balance = balance + 100');
  // Transaction open for seconds or minutes
});

// ✅ Keep transactions short
const balance = await db.query('SELECT balance FROM accounts');
await fetch('https://payment-provider.com/charge'); // External call outside transaction
await db.transaction(async (tx) => {
  await tx.query('UPDATE accounts SET balance = balance - 100');
  await tx.query('UPDATE accounts SET balance = balance + 100');
});
```

### 2. Catching and Swallowing Errors

```javascript
// ❌ Transaction marked for rollback, but code continues
await db.transaction(async (tx) => {
  try {
    await tx.query('INSERT INTO orders ...');
  } catch (err) {
    console.log('Insert failed, continuing...');
    // Transaction is now in error state
    // Any further queries will fail
  }
  await tx.query('UPDATE inventory ...'); // This fails too!
});

// ✅ Let errors propagate
await db.transaction(async (tx) => {
  await tx.query('INSERT INTO orders ...');
  await tx.query('UPDATE inventory ...');
}); // If either fails, entire transaction rolls back
```

### 3. Nested Transactions That Aren't

```sql
-- PostgreSQL doesn't have true nested transactions
BEGIN;
  INSERT INTO orders VALUES (1);
  BEGIN;  -- This is a SAVEPOINT, not a new transaction
    INSERT INTO order_items VALUES (1, 1);
  COMMIT;
  INSERT INTO payments VALUES (1);
COMMIT;
```

Use savepoints explicitly:

```sql
BEGIN;
  INSERT INTO orders VALUES (1);
  SAVEPOINT order_items;
  INSERT INTO order_items VALUES (1, 1);
  -- If this fails:
  ROLLBACK TO order_items;  -- Go back to savepoint, transaction continues
  INSERT INTO payments VALUES (1);
COMMIT;
```

## Quick Reference

| Scenario | Isolation Level | Notes |
|----------|----------------|-------|
| Web app read-mostly | Read Committed | Default in Postgres, good balance |
| Financial transfers | Serializable | Prevent all anomalies, handle retries |
| Reporting queries | Repeatable Read | Consistent snapshot for long reads |
| Analytics batch jobs | Read Committed | Usually fine, less locking |
| High-concurrency writes | Read Committed + optimistic locking | Avoid serialization failures |

## Actionable Takeaways

1. **Wrap related operations in transactions.** Any operation that spans multiple tables or rows should be atomic.

2. **Use the lowest isolation level that works.** Serializable is safest but slowest. Start with Read Committed.

3. **Lock in consistent order.** Sort IDs, lock parent before child—prevent deadlocks by design.

4. **Keep transactions short.** No external API calls, no user input waiting. Milliseconds, not seconds.

5. **Handle serialization failures.** With higher isolation levels, retries become necessary. Plan for it.

6. **Use `FOR UPDATE` when reading before writing.** Prevents concurrent modifications to rows you're about to change.

7. **Test with concurrent load.** Race conditions and deadlocks only appear under concurrent stress. Use tools like `pgbench` or write concurrent test scripts.