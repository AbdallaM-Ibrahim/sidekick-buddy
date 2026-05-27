# Database Schema Blueprints

This document outlines the Drizzle ORM + PostgreSQL schema design for Sidekick Buddy, incorporating manual asset price tracking.

## Core Entities

### 1. `users`
- `id`: string (uuid) / primary key
- `email`: string (unique)
- `name`: string
- `created_at`: timestamp
- *(BetterAuth will auto-generate its required fields here as well)*

### 2. `user_profiles` (Auth metadata extension)
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `default_currency`: string (e.g. "USD")
- `theme`: string (e.g. "DARK")
- `streak_count`: integer (current consecutive days logged)
- `last_logged_date`: date (YYYY-MM-DD)

### 3. `wallets` (Generic Assets/Accounts/Vaults)
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `name`: string (e.g., "Main Bank", "Gold Stash", "Crypto Wallet")
- `type`: enum ("CASH", "BANK", "CREDIT", "INVESTMENT", "CUSTOM_ASSET")
- `currency_or_unit`: string (e.g., "USD", "EUR", "GRAMS", "BTC")
- `current_balance`: real / integer (stored in lowest denominator or base unit quantity)
- `created_at`: timestamp

### 4. `asset_prices` (Manual Asset Price History Log)
*Stores historical pricing values of non-fiat currencies or assets manually input by the user.*
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `unit`: string (e.g., "GRAMS" [for gold], "BTC", "AAPL")
- `price_in_base`: integer (stored in cents, representing the value in user's base currency, e.g. USD)
- `recorded_at`: timestamp

### 5. `spaces` (Contexts like Personal, Family)
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `name`: string (e.g., "Personal", "Family")
- `icon`: string
- `color`: string

### 6. `tags` (Flat Categories like Groceries, Entertainment)
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `name`: string
- `color`: string

### 7. `transaction_tags` (Many-to-Many Join Table)
- `transaction_id`: string (fk -> transactions.id)
- `tag_id`: string (fk -> tags.id)

### 8. `budgets`
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `space_id`: string (fk, nullable - applies to the whole space if set)
- `tag_id`: string (fk, nullable - applies to a specific tag if set)
- `amount`: integer (cents)
- `reset_cron`: string (e.g., '0 0 * * 0' for weekly resets. The budget cycle is entirely defined by this cron expression, replacing rigid start/end dates.)

### 9. `transactions`
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `wallet_id`: string (fk -> wallets.id)
- `space_id`: string (fk -> spaces.id)
- `type`: enum ("INCOME", "EXPENSE", "TRANSFER")
- `amount`: integer (cents or quantity unit cents)
- `date`: timestamp (when the transaction occurred)
- `note`: text (optional)
- `is_periodic_instance`: boolean
- `periodic_id`: string (fk -> periodic_transactions.id, nullable)

### 10. `periodic_transactions` (The Modular Cron Feature)
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `wallet_id`: string (fk -> wallets.id)
- `space_id`: string (fk -> spaces.id)
- `amount`: integer
- `type`: enum ("INCOME", "EXPENSE")
- `cron_expression`: string (kept for UI editing, but BullMQ handles the actual execution)
- `job_id`: string (links to the Redis repeatable job)
- `is_active`: boolean
- `auto_commit`: boolean

### 11. `goals`
- `id`: string (uuid) / pk
- `user_id`: string (fk -> users.id)
- `name`: string (e.g., "Summer Trip")
- `target_amount`: integer
- `current_amount`: integer
- `deadline_at`: timestamp (nullable)
- `description_rich`: text (markdown or rich text for attachments/details)

### 12. `daily_account_stats` (Postgres Materialized View)
*Used to instantly load dashboard charts without querying millions of transactions.*
- Defined natively in PostgreSQL via `CREATE MATERIALIZED VIEW`.
- Refreshed asynchronously via BullMQ Cron Jobs.
- `wallet_id`: string (fk -> wallets.id)
- `date`: date (YYYY-MM-DD)
- `total_income`: integer
- `total_expense`: integer
- `closing_balance`: integer

## Relationships / Drizzle Outline
- A **User** has many Wallets, Spaces, Tags, Transactions, Budgets, Goals, and Asset Prices.
- A **Wallet** has many Transactions.
- A **Transaction** belongs to one Space and has many Tags (via `transaction_tags`).
- A **Goal** belongs to one User.
- A **Budget** applies to a Space and/or Tag.
- A **Periodic Transaction** spawns many normal Transactions.
