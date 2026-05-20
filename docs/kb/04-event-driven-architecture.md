# Event-Driven Architecture in Bun + Nitro

You asked how an Event-Driven Architecture (EDA) can be implemented within our chosen stack (Bun + TanStack Start + Nitro), and asked me to challenge it with DIY hacks.

## How it works conceptually
Instead of the `Transactions` module importing the `Gamification` module to update a streak, the `Transactions` module simply shouts into the void: *"Hey, a transaction was just created!"*
Other modules (Gamification, Goals, Notifications) listen for that specific shout and react independently.

## Implementation: BullMQ & Redis

Because we have integrated **Redis** and **BullMQ** into our stack, we do not need to rely on fragile in-memory event emitters or messy database polling. BullMQ perfectly solves event-driven architecture out of the box.

### How it works
1. **The Producers**: When the `Transactions` module finishes creating an entry in the database, it simply drops an event payload into a Redis Queue: 
   `gamificationQueue.add('transaction.created', { amount, userId })`.
2. **The Workers (Listeners)**: The `Gamification` module spins up a BullMQ Worker listening to `gamificationQueue`. As soon as the event hits Redis, the worker picks it up and increments the streak.

### Why this is the ultimate solution:
- **True Decoupling:** The Transaction module doesn't care if Gamification fails, crashes, or is slow.
- **Guaranteed Execution:** If the Gamification worker crashes while updating a streak, BullMQ automatically retries the job. Events are never lost because they are persisted in Redis.
- **Infinite Scalability:** Whether you run 1 server or 100 servers, Redis handles the event distribution perfectly without any race conditions.

## Reusing BetterAuth Schema
BetterAuth provides standard tables (`user`, `session`, `account`, `verification`). We will **not** touch these tables. Instead, any custom user data (like user preferences, default currency) will live in a `user_profiles` table that has a `user_id` Foreign Key referencing the BetterAuth `user` table. This prevents any clashing and keeps auth isolated.
