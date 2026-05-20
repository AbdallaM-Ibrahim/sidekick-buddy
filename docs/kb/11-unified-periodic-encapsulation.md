# The Unified Periodic Encapsulation Debate

This document explores the architectural challenge of handling recurring events in the application. We have two distinct domain entities (`Budgets` and `Periodic Transactions`) that require cron-based scheduling. 

How do we build a system that manages time and schedules efficiently without bleeding domain logic (like what a "budget" is) into the background job runner?

---

## Part 1: Analyzing the Domain Differences

Before building a unified engine, we must understand how wildly different the consumers are.

### `periodic_transactions` Properties
- **Human-Centric:** A periodic transaction trigger does not silently mutate the database; it generates a `PENDING` state that requires user confirmation.
- **Mutable Timelines:** A user can "Pay Early", meaning the schedule is disrupted and the *next* trigger must be skipped.
- **Undoable:** If a user confirms by mistake, the action can be undone, which may require resetting the state of the scheduled instance.
- **Notifications:** Inherently tied to a delayed 24-hour notification system.

### `budgets` Properties
- **System-Centric (Silent):** A budget reset happens invisibly in the background. It requires absolutely no human interaction.
- **Immutable Timelines:** A budget interval (e.g., Weekly) resets at exactly midnight on Sunday. A user cannot "Reset Budget Early" in a way that skips the next Sunday.
- **State Rollover:** It involves querying the past period's total spending, archiving it, and resetting the `current_spent` tracker to 0.

**The Conflict:** If we write a single background worker that handles both, the worker's code will become a massive `if/else` nightmare of domain logic. Therefore, we must **encapsulate the periodic logic** (the "alarm clock") away from the domain logic (the "action").

---

## Part 2: Implementation Approaches

Here are three distinct architectural approaches to implement a unified, encapsulated periodic engine.

### Approach A: The Polymorphic `schedules` Table (Strict Database Decoupling)

In this approach, we create a central SQL table to manage all time, regardless of what domain owns it.

**Step 1: Database Schema Alterations**
- Create a `schedules` table: `id`, `consumer_id` (string), `consumer_type` (enum: "BUDGET", "TRANSACTION"), `cron_expression`, `last_run_at`.
- Remove `cron_expression` from both `budgets` and `periodic_transactions`. Instead, they just hold a `schedule_id` foreign key.

**Step 2: Engine Registration**
When a Budget is created, the Budget Service inserts a row into `schedules`. The Periodic Engine only ever monitors the `schedules` table. It registers a BullMQ repeatable job for that `schedule_id`.

**Step 3: Execution & Routing**
When BullMQ fires, the Periodic Worker looks up the `schedule_id`. It sees the `consumer_type` is `"BUDGET"`. It emits a generic event to the Event Bus: `EventBus.emit('schedule.triggered', { type: 'BUDGET', id: 'budget_123' })`.

**Pros:**
- Complete normalization of time. All cron logic lives in exactly one database table.
- Easy to query: `SELECT * FROM schedules WHERE next_run < NOW()` gives you a unified view of everything happening in the app.

**Cons:**
- **Polymorphic Pain:** SQL databases hate polymorphic associations. `consumer_id` cannot be a strict Foreign Key because it might point to `budgets` OR `periodic_transactions`. Data integrity goes out the window. If a budget is deleted, the schedule row might be orphaned unless application-level cleanup is flawless.

---

### Approach B: API-Driven Registration (The Redis Source-of-Truth)

In this approach, the PostgreSQL database is kept strictly domain-focused. There is no central `schedules` table. Instead, the Periodic Module is purely a TypeScript Service wrapping BullMQ.

**Step 1: Database Schema Alterations**
- Keep `cron_expression` and `bullmq_job_id` directly on the `budgets` and `periodic_transactions` tables.

**Step 2: The Periodic API Contract**
We build a generic service:
```typescript
class PeriodicEngine {
  static async registerCron(consumerId: string, topic: string, cron: string) {
    const job = await genericQueue.add('trigger', { consumerId, topic }, { repeat: { cron } });
    return job.id; // Returns to the domain to be saved as `bullmq_job_id`
  }
}
```

**Step 3: Execution & Routing**
When BullMQ evaluates the cron, it runs the `genericQueue` worker. The worker reads the payload (e.g., `{ consumerId: 'budget_1', topic: 'BUDGET_RESET' }`). It fires the event to the Event Bus.

**Step 4: Handling "Skips" (Early Resolution)**
Because the domains own their data, the `Transactions` module can call `PeriodicEngine.skipNext(bullmq_job_id)`. The Engine interacts directly with Redis to bypass the next iteration, without needing to update a central SQL table.

**Pros:**
- Strict SQL Foreign Keys are preserved. Domains own their data completely.
- Extremely scalable. BullMQ/Redis acts as the central router effortlessly.

**Cons:**
- **Sync Issues:** If your Redis instance is completely wiped (disaster recovery), you lose all your scheduled jobs. You would have to write a boot-script that scans the `budgets` and `periodic_transactions` tables to re-register the `cron_expression`s into BullMQ.

---

### Approach C: Dedicated Domain Queues (Zero Centralization)

In this approach, we reject the idea of a "Unified Periodic Engine" entirely. Instead, we treat BullMQ as a dumb utility, and every domain implements its own cron worker.

**Step 1: Setup**
- `Budgets` module spins up `budget_queue` and `BudgetWorker`.
- `Transactions` module spins up `transaction_queue` and `TransactionWorker`.

**Step 2: Execution**
- When a budget is created, it registers a job directly on the `budget_queue`. When it fires, the `BudgetWorker` handles the logic natively. No Event Bus is used.

**Pros:**
- The absolute simplest code to write. No abstract APIs, no generic payloads, no event buses.
- Complete domain isolation. If the Budget worker crashes, the Transactions worker keeps running.

**Cons:**
- **Code Duplication:** The logic for "How do I skip a cron job?" or "How do I ensure this didn't run twice?" has to be written in the Budget module AND the Transactions module.
- **Global Operations:** If you want a master switch in the admin panel to "Pause all background tasks", you have to manually loop through and pause 5 different queues instead of one central Periodic Engine.

---

## Conclusion & Recommendation
For a modern, highly decoupled architecture using Bun and TanStack Start, **Approach B (API-Driven Registration)** is the industry standard. It maintains strict SQL integrity, perfectly encapsulates the generic concept of time into a single service, and leverages the Event Bus to keep modules blissfully ignorant of each other. 

The single drawback (Redis state loss) is easily solved by a one-off `syncSchedules()` function that runs on server boot to ensure BullMQ matches the SQL database.
