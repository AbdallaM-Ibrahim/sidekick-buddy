# Minimal Database Indexes Guide

*Agent Note: This document outlines the absolute minimum PostgreSQL indexes required to ensure Sidekick Buddy remains lightning-fast when querying millions of rows. It does not include Primary Keys (PKs), as Drizzle/Postgres indexes those automatically.*

## 1. `idx_transactions_user_date`
**Definition:** `CREATE INDEX idx_transactions_user_date ON transactions (user_id, date DESC);`
**Why it's necessary:** 
The main dashboard will almost always query: "Get all transactions for `user_id = X` ordered by `date` descending" (or within a specific month). A composite index on `(user_id, date)` allows Postgres to instantly jump to the user's records and read them in chronological order without sorting millions of rows in memory.

## 2. `idx_transactions_wallet`
**Definition:** `CREATE INDEX idx_transactions_wallet ON transactions (wallet_id, date DESC);`
**Why it's necessary:**
When a user clicks on their "Main Bank Vault" to see its history, the app queries: `WHERE wallet_id = Y ORDER BY date DESC`. Without this, Postgres would have to scan every transaction the user ever made just to find the ones belonging to this specific wallet.

## 3. `idx_transactions_space`
**Definition:** `CREATE INDEX idx_transactions_space ON transactions (space_id);`
**Why it's necessary:**
Since spaces act as large buckets (e.g., "Family", "Personal"), users will frequently filter their analytics by a Space. This index ensures that `WHERE space_id = Z` does not require a full table scan.

## 4. The `tags` Index (Depends on chosen architecture)
**If using the Join Table (`transaction_tags`):**
- `CREATE INDEX idx_tt_tag ON transaction_tags (tag_id);`
- *Why:* To quickly find all transactions associated with a specific tag (the reverse lookup is covered by the PK of the join table).

**If using Postgres Arrays (`tags: text[]`):**
- `CREATE INDEX idx_transactions_tags_gin ON transactions USING GIN (tags);`
- *Why:* A standard B-Tree index cannot search inside arrays. A Generalized Inverted Index (GIN) maps every individual tag inside the arrays back to the row, making queries like `WHERE 'groceries' = ANY(tags)` instant.

## 5. `idx_budgets_lookup`
**Definition:** `CREATE UNIQUE INDEX idx_budgets_lookup ON budgets (user_id, space_id, tag_id);`
**Why it's necessary:**
When validating if a transaction exceeds a budget, the system needs to quickly look up if a budget exists for that specific user + space + tag combination. Making it a `UNIQUE` index also enforces data integrity (preventing two duplicate budget rules from being created).

## 6. `idx_asset_prices_lookup`
**Definition:** `CREATE INDEX idx_asset_prices_lookup ON asset_prices (user_id, unit, recorded_at DESC);`
**Why it's necessary:**
To dynamically calculate non-fiat asset valuations, the system will regularly query the latest manually entered price: `WHERE user_id = X AND unit = Y ORDER BY recorded_at DESC LIMIT 1`. This composite index makes this $O(1)$ query near instantaneous.

---

## Summary
By adding just these 6 indexes, the database will handle dashboard queries, manual asset price evaluations, and analytical checks on millions of rows with near 0ms latency. Anything more than this is premature optimization.
