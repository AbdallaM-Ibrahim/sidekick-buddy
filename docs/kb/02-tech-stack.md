# Tech Stack & Architecture Keynote

## The Final Stack

- **Runtime & Package Manager**: Bun
- **Framework**: TanStack Start (React) powered by Nitro
- **Database**: PostgreSQL
- **Message Queue & Events**: Redis + BullMQ
- **ORM**: Drizzle ORM
- **Auth**: BetterAuth

---

## Keynote: Pros, Cons, and My Thoughts

**The Pros:**
1. **Enterprise Grade Reliability**: PostgreSQL is the gold standard for relational data. It handles concurrent writes natively, provides advanced features like `JSONB` for flexible metadata, and guarantees strict constraints essential for financial data.
2. **Robust Asynchronous Processing**: Redis + BullMQ completely solves event-driven architecture, cron jobs, and delayed tasks (like notifications) out-of-the-box. We get retries, exponential backoffs, and guaranteed execution without writing boilerplate.
3. **Extreme Type Safety**: TanStack Start + Drizzle ORM means your types flow perfectly from your PostgreSQL database all the way to your React components.
4. **Bleeding Edge DX**: Bun is incredibly fast, and TanStack Start brings the best routing and data-fetching paradigms (via TanStack Query) to the React ecosystem.

**The Cons & Risks:**
1. **Infrastructure Complexity**: You are no longer dealing with a single local file. You must run and maintain a Postgres container and a Redis container. 
2. **TanStack Start is Beta**: It is relatively new. You might run into bugs, missing documentation, or breaking changes in minor updates.

---

## Analytics Challenge: Millions of Entries

**Your Question:** How do we optimize for statistics (daily/weekly/monthly) with millions of entries and custom ranges?

**My Assessment & Solution:**
Since we are using PostgreSQL, we don't need to reinvent the wheel with DIY cron-based rollup tables. We use **Materialized Views**.

1. **The Materialized View**: We create a `MATERIALIZED VIEW daily_wallet_stats AS SELECT ...` in Postgres that automatically aggregates total income and expenses per day.
2. **The Refresh Worker**: We configure a BullMQ Repeatable Job (Cron) to run `REFRESH MATERIALIZED VIEW CONCURRENTLY daily_wallet_stats` every night at midnight.
3. **Querying**: 
   - When the dashboard loads, it queries the materialized view (which acts like a cached, pre-calculated table).
   - This keeps the app lightning fast, easily handling millions of transactions, while delegating the heavy computational lifting entirely to the database engine.
