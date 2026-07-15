# Complete Example: Simple Banking

## Goal

Process deposits and withdrawals, track balances, enforce overdraft limits.

## Schema

```yaml
# schema.yaml
type: object
properties:
  transaction_id:
    type: string
  account_id:
    type: string
  transaction_type:
    type: string
    enum: [deposit, withdrawal]
  amount:
    type: number
  status:
    type: string
    enum: [completed, rejected]
  reason:
    type: string
  balance_before:
    type: number
  balance_after:
    type: number
  timestamp:
    type: string
    format: date-time

required: [transaction_id, account_id, status]
```

## Derivation

```yaml
# flow.yaml
collections:
  acmeCo/banking/transaction-results:
    schema: schema.yaml
    key: [/transaction_id]
    derive:
      using:
        sqlite:
          migrations:
            - |
              CREATE TABLE accounts (
                account_id TEXT PRIMARY KEY NOT NULL,
                balance REAL NOT NULL DEFAULT 0,
                overdraft_limit REAL NOT NULL DEFAULT 0
              );
      transforms:
        - name: processTransaction
          source: acmeCo/banking/transactions
          shuffle: { key: [/account_id] }
          lambda: processTransaction.sql
```

## SQL: processTransaction.sql

```sql
-- Get current account state
WITH account AS (
  SELECT
    COALESCE(balance, 0) AS balance,
    COALESCE(overdraft_limit, 0) AS overdraft_limit
  FROM (SELECT 1) LEFT JOIN accounts ON account_id = $account_id
),
new_balance AS (
  SELECT
    CASE $type
      WHEN 'deposit' THEN account.balance + $amount
      WHEN 'withdrawal' THEN account.balance - $amount
    END AS balance
  FROM account
),
can_proceed AS (
  SELECT
    CASE
      WHEN $type = 'deposit' THEN 1
      WHEN $type = 'withdrawal' AND new_balance.balance >= -account.overdraft_limit THEN 1
      ELSE 0
    END AS allowed
  FROM account, new_balance
)
SELECT
  $transaction_id AS transaction_id,
  $account_id AS account_id,
  $type AS transaction_type,
  $amount AS amount,
  CASE WHEN can_proceed.allowed THEN 'completed' ELSE 'rejected' END AS status,
  CASE
    WHEN can_proceed.allowed THEN 'Transaction processed'
    ELSE 'Insufficient funds (overdraft limit: ' || account.overdraft_limit || ')'
  END AS reason,
  account.balance AS balance_before,
  CASE
    WHEN can_proceed.allowed THEN new_balance.balance
    ELSE account.balance
  END AS balance_after,
  $timestamp AS timestamp
FROM account, new_balance, can_proceed;

-- Update balance if transaction succeeded
INSERT INTO accounts (account_id, balance, overdraft_limit)
SELECT
  $account_id,
  CASE $type WHEN 'deposit' THEN $amount ELSE -$amount END,
  0
WHERE (
  SELECT
    CASE
      WHEN $type = 'deposit' THEN 1
      WHEN (SELECT COALESCE(balance, 0) FROM accounts WHERE account_id = $account_id) - $amount
           >= -(SELECT COALESCE(overdraft_limit, 0) FROM accounts WHERE account_id = $account_id)
      THEN 1
      ELSE 0
    END
)
ON CONFLICT(account_id) DO UPDATE SET
  balance = balance + CASE $type WHEN 'deposit' THEN $amount ELSE -$amount END;
```

## Workflow

```bash
# Publish
flowctl catalog publish --source flow.yaml --auto-approve

# Check status
flowctl catalog status acmeCo/banking/transaction-results

# Read transaction results
flowctl collections read --collection acmeCo/banking/transaction-results --uncommitted | \
  jq 'select(.account_id == "acct_123")'
```

## Expected Output

```json
{
  "transaction_id": "txn_001",
  "account_id": "acct_123",
  "transaction_type": "deposit",
  "amount": 1000.00,
  "status": "completed",
  "reason": "Transaction processed",
  "balance_before": 0,
  "balance_after": 1000.00,
  "timestamp": "2025-12-20T10:00:00Z"
}
{
  "transaction_id": "txn_002",
  "account_id": "acct_123",
  "transaction_type": "withdrawal",
  "amount": 1500.00,
  "status": "rejected",
  "reason": "Insufficient funds (overdraft limit: 0)",
  "balance_before": 1000.00,
  "balance_after": 1000.00,
  "timestamp": "2025-12-20T10:05:00Z"
}
```
