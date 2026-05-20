# Generic Periodic Engine Implementation

*This document outlines the encapsulated Periodic Module. This module is domain-agnostic; it has absolutely zero knowledge of Wallets, Transactions, or Budgets. Its sole responsibility is managing generic time-based schedules, ensuring idempotency, and notifying external modules when a period cycle occurs.*

## 1. Core Philosophy: The Decoupled Engine
The Periodic Engine is a standalone service wrapping BullMQ. 
Modules (like `Budgets` or `Transactions`) act as **Consumers**. When a Consumer needs something to happen on a schedule, it delegates the scheduling responsibility to the Periodic Engine.
The Engine does not write to financial tables. It simply acts as a highly reliable alarm clock that guarantees a "wake up" call happens exactly once per scheduled cycle.

## 2. Engine Registration (The API Contract)
When a consumer (e.g., the Budget Module) wants to set a weekly reset, it calls the Periodic Engine's API:

```typescript
PeriodicEngine.registerSchedule({
  consumerId: "budget_12345", // The ID of the row in the consumer's table
  consumerType: "BUDGET_RESET", // The topic the engine will broadcast
  cronExpression: "0 0 * * 0",
});
```

**Internal Engine Action:**
1. The Engine generates a unique `jobId` in BullMQ linking to this schedule.
2. The Engine may optionally store this schedule in a generic `schedules` database table (if we choose Approach 1 of the encapsulation debate) or just rely strictly on BullMQ's internal Redis state.

## 3. The Generic Trigger Phase (BullMQ Worker)
When BullMQ determines it is time to fire based on the cron expression, the generic Periodic Worker wakes up.
1. It reads the job payload: `{ consumerId: "budget_12345", consumerType: "BUDGET_RESET", expectedCycleTime: "2026-05-13T00:00:00Z" }`.
2. **Idempotency Check**: The Engine ensures it has not already fired an event for this exact `consumerId` and `expectedCycleTime`. It might check an `event_logs` table to confirm.
3. **Event Broadcasting**: The Engine screams into the void (via the Event Bus):
   `EventBus.emit("periodic:triggered", payload)`
4. The Engine goes back to sleep. It does not know what a budget is. It does not update any wallet. It has done its job.

## 4. Consumer Reactions (How Domains Hook In)
The external modules are listening to the Event Bus for the `periodic:triggered` event.

### Example A: The Budget Consumer
The `Budgets` module hears `periodic:triggered`. 
It checks the payload. If `consumerType === "BUDGET_RESET"`:
1. It looks up `budget_12345` in its own database table.
2. It executes its domain logic: Calculating rollover amounts, closing out the previous cycle, and initializing the spending tracking for the new week.

### Example B: The Periodic Transaction Consumer
The `Transactions` module hears `periodic:triggered`.
If `consumerType === "PERIODIC_TRANSACTION"`:
1. It looks up the blueprint in the `periodic_transactions` table.
2. It generates a new transaction row with `status = PENDING`.
3. It queues a delayed notification.

## 5. Handling Edge Cases: "Early Execution"
What if the user clicks "Pay Early" on a transaction, meaning the consumer wants to skip the next scheduled engine trigger?
The Consumer must tell the Engine to skip the next beat:
```typescript
PeriodicEngine.registerSkip({
  consumerId: "transaction_999",
  skipCycle: "2026-05-15T00:00:00Z" // The date of the trigger to ignore
});
```
When the Engine fires on May 15th, it checks its "Skip Registry". It sees `transaction_999` is flagged to skip this cycle. The Engine aborts and does *not* broadcast the event.

## Summary
By encapsulating the periodic logic into a generic Engine, we never rewrite cron logic. If we add a new feature later (e.g., "Weekly Goal Reminders" or "Monthly Data Export"), we simply create a new Consumer that calls `PeriodicEngine.registerSchedule()`. The background plumbing is solved once, perfectly.
