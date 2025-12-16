# cf-workflow-rollback

Saga-pattern rollback utility for [Cloudflare Workflows](https://developers.cloudflare.com/workflows/).

## Installation

```bash
npm install cf-workflow-rollback
# or
bun add cf-workflow-rollback
```

## Usage

```typescript
import { WorkflowEntrypoint, type WorkflowEvent, type WorkflowStep } from "cloudflare:workers";
import { withRollback } from "cf-workflow-rollback";

export class MyWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, workflowStep: WorkflowStep) {
    const step = withRollback(workflowStep);

    try {
      // Regular step (no rollback needed)
      const data = await step.do("fetch data", async () => {
        return fetchExternalData();
      });

      // Step with rollback - if a later step fails, this will be undone
      const id = await step.doWithRollback("save to database", {
        run: async () => {
          return await db.insert(data);
        },
        undo: async (error, id) => {
          await db.delete(id);
        },
      });

      // Another step with rollback
      await step.doWithRollback("charge payment", {
        run: async () => {
          return await payments.charge(event.payload.amount);
        },
        undo: async (error, chargeId) => {
          await payments.refund(chargeId);
        },
      });

      // Final step (no rollback - if this fails, previous steps are undone)
      await step.do("send confirmation", async () => {
        await sendEmail(event.payload.email);
      });

    } catch (error) {
      // Rollback all successful steps in reverse order
      await step.rollbackAll(error);

      // Optionally: cleanup steps that always run on failure
      await step.do("notify failure", async () => {
        await sendFailureNotification();
      });

      throw error;
    }
  }
}
```

## API

### `withRollback(workflowStep: WorkflowStep)`

Wraps a Cloudflare Workflow step with rollback capabilities.

Returns an object with:

| Method | Description |
|--------|-------------|
| `do(name, callback)` | The original `step.do` method |
| `do(name, config, callback)` | The original `step.do` method with config |
| `doWithRollback(name, handler, config?)` | Execute a step with an undo handler |
| `rollbackAll(error)` | Execute all registered undo handlers in LIFO order |

### `RollbackHandler<T>`

```typescript
type RollbackHandler<T> = {
  run: () => Promise<T>;
  undo: (err: unknown, value: T) => Promise<void>;
};
```

- `run` - The step function to execute
- `undo` - Called with the error and the return value of `run` if a later step fails

## How It Works

This utility implements the [Saga pattern](https://microservices.io/patterns/data/saga.html) for Cloudflare Workflows:

1. Each `doWithRollback` step registers an undo handler after successful execution
2. Undo handlers are stored in a LIFO (last-in-first-out) stack
3. On failure, `rollbackAll` executes undo handlers in reverse order
4. Each undo operation is wrapped in `step.do` for durability and retry

### Replay Safety

The undo stack is correctly rebuilt on workflow replay because:

- `step.do` returns cached results for completed steps
- Undo handlers are registered after each successful step
- The same sequence of steps produces the same undo stack

## License

MIT
