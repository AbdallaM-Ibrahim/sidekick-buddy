# Database Schema Blueprints

This document outlines the initial Drizzle ORM + PostgreSQL schema design for Sidekick Buddy.

## Core Entities

### 1. `users`
- `id`: string (uuid) / primary key
- `email`: string (unique)
- `name`: string
- `created_at`: timestamp
- *(BetterAuth will auto-generate its required fields here as well)*

### 2. `wallets` (Generic Assets/Accounts/Vaults)
- `id`: string (uuid) / pk
- `user_id`: string (fk)
- `name`: string (e.g., "Main Bank", "Gold Stash", "Crypto Wallet")
- `type`: enum ("CASH", "BANK", "CREDIT", "INVESTMENT", "CUSTOM_ASSET")
- `currency_or_unit`: string (e.g., "USD", "EUR", "GRAMS", "BTC")
- `current_balance`: real / integer (stored in lowest denominator, e.g. cents)
- `created_at`: timestamp

### 3. `spaces` (Contexts like Personal, Family)
- `id`: string (uuid) / pk
- `user_id`: string (fk)
- `name`: string (e.g., "Personal", "Family")
- `icon`: string
- `color`: string

### 4. `tags` (Flat Categories like Groceries, Entertainment)
- `id`: string (uuid) / pk
- `user_id`: string (fk)
- `name`: string
- `color`: string

### 5. `transaction_tags` (Many-to-Many Join Table)
- `transaction_id`: string (fk)
- `tag_id`: string (fk)

### 6. `budgets`
- `id`: string (uuid) / pk
- `user_id`: string (fk)
- `space_id`: string (fk, nullable - applies to the whole space if set)
- `tag_id`: string (fk, nullable - applies to a specific tag if set)
- `amount`: integer (cents)
- `reset_cron`: string (e.g., '0 0 * * 0' for weekly resets. The budget cycle is entirely defined by this cron expression, replacing rigid start/end dates.)

### 7. `transactions`
- `id`: string (uuid) / pk
- `user_id`: string (fk)
- `wallet_id`: string (fk)
- `space_id`: string (fk)
- `type`: enum ("INCOME", "EXPENSE", "TRANSFER")
- `amount`: integer (cents)
- `date`: timestamp (when the transaction occurred)
- `note`: text (optional)
- `is_periodic_instance`: boolean
- `periodic_id`: string (fk, nullable)

### 8. `periodic_transactions` (The Modular Cron Feature)
- `id`: string (uuid) / pk
- `user_id`: string (fk)
- `wallet_id`: string (fk)
- `space_id`: string (fk)
- `amount`: integer
- `type`: enum ("INCOME", "EXPENSE")
- `cron_expression`: string (kept for UI editing, but BullMQ handles the actual execution)
- `job_id`: string (links to the Redis repeatable job)
- `is_active`: boolean
- `auto_commit`: boolean

### 9. `goals`
- `id`: string (uuid) / pk
- `user_id`: string (fk)
- `name`: string (e.g., "Summer Trip")
- `target_amount`: integer
- `current_amount`: integer
- `deadline_at`: timestamp (nullable)
- `description_rich`: text (markdown or rich text for attachments/details)

### 10. `daily_account_stats` (Postgres Materialized View)
*Used to instantly load dashboard charts without querying millions of transactions.*
- Defined natively in PostgreSQL via `CREATE MATERIALIZED VIEW`.
- Refreshed asynchronously via BullMQ Cron Jobs.
- `wallet_id`: string (fk)
- `date`: date (YYYY-MM-DD)
- `total_income`: integer
- `total_expense`: integer
- `closing_balance`: integer

## Relationships / Drizzle Outline
- A User has many Wallets, Spaces, Tags, Transactions, and Goals.
- A Wallet has many Transactions.
- A Transaction belongs to one Space and has many Tags.
- A Budget applies to a Space and/or Tag.
- A Periodic Transaction spawns many normal Transactions.
