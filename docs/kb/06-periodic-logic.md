# Periodic Logic & Notifications

The periodic transaction engine needs to be robust, extendable, and handle human delays.

## The Logic Flow
1. **Time Library**: We will use `date-fns` (or `dayjs`). They are lightweight, tree-shakeable, and handle complex timezone logic natively.
2. **The Cron Engine**: We use a **BullMQ Repeatable Job**. A worker is registered to fire every hour (e.g. `cron: '0 * * * *'`). It queries the `periodic_transactions` table for anything where `next_run_at <= NOW()`.

## Early / Delayed Resolution & Undo
When a periodic transaction triggers, it does **not** silently deduct money from your wallet (unless you explicitly set `auto_commit = true`). 
Instead, it creates a "Pending" transaction instance.

### The States of a Periodic Instance:
- **Pending**: It's due today (or past due). It shows up in a loud "Action Required" widget on the dashboard.
- **Resolved**: You click "Confirm" -> It officially deducts the money and updates the `wallets` balance.
- **Resolved Early**: You click "Pay Now" on a bill due next week. The system creates the instance early and pushes the `next_run_at` forward.
- **Undo Logic**: If you accidentally hit Confirm, an "Undo" button simply deletes the instance, restores the wallet balance (via a reverse transaction or direct update), and resets the instance back to "Pending".

## Notifications (Powered by BullMQ)
Instead of constantly polling the database to see if an instance is overdue, we use **BullMQ Delayed Jobs**:
- When a "Pending" instance is created, we immediately add a job to the `notificationsQueue` with a delay of 24 hours: 
  `notificationsQueue.add('periodic:overdue', { instanceId }, { delay: 24 * 60 * 60 * 1000 })`.
- If the user clicks "Confirm" *before* 24 hours pass, we simply remove the delayed job from the queue: 
  `notificationsQueue.remove(jobId)`.
- If they don't confirm it, 24 hours later BullMQ automatically fires the job, and the Notification Worker sends the push notification or email via Resend.
