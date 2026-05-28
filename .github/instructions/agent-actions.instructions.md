---
description: "Use when adding, modifying, or reviewing a nanobrowser agent action (click, goToUrl, scroll, dropdown, etc.) in chrome-extension/src/background/agent/actions. Covers Zod schemas, the Action class, ActionBuilder registration, ActionResult contract, and event emission."
applyTo: "chrome-extension/src/background/agent/actions/**, chrome-extension/src/background/agent/agents/navigator.ts"
---

# Agent Action Conventions

Actions are the building blocks the Navigator agent calls to interact with the page. Every action follows the same three-part contract: **schema → handler → registration**.

## 1. Schema (Zod, in [actions/schemas.ts](chrome-extension/src/background/agent/actions/schemas.ts))

Export an `ActionSchema` named `<name>ActionSchema`:

```ts
export const clickElementActionSchema: ActionSchema = {
  name: 'click_element',
  description: 'Click an interactive element by its index',
  schema: z.object({
    index: z.number().int().describe('Index of the clickable element'),
    xpath: z.string().optional().describe('Optional xpath for verification'),
  }),
};
```

Rules:
- Use `snake_case` for the `name` — it is the function name the LLM emits.
- Use `.describe(...)` on every Zod field. The string is rendered into the LLM prompt by `Action.prompt()` ([actions/builder.ts](chrome-extension/src/background/agent/actions/builder.ts)).
- Mark optional fields with `.optional()`; the prompt rendering relies on `isOptional()`.
- If the action targets a clickable element, name the field `index` and pass `hasIndex: true` when constructing the `Action` (used by `getIndexArg`).

## 2. Handler (in [actions/builder.ts](chrome-extension/src/background/agent/actions/builder.ts))

Handlers are `async` functions returning `ActionResult` from [`agent/types`](chrome-extension/src/background/agent/types.ts):

```ts
new Action(
  async (input: z.infer<typeof clickElementActionSchema.schema>) => {
    // ... do work via BrowserContext / BrowserPage
    return new ActionResult({
      isDone: false,
      extractedContent: t('act_click_ok', [String(input.index), label]),
      includeInMemory: true,
    });
  },
  clickElementActionSchema,
  /* hasIndex */ true,
);
```

Handler rules:
- All user-visible strings (`extractedContent`, error messages) go through `t(...)` with `act_*` keys — see [i18n.instructions.md](.github/instructions/i18n.instructions.md).
- Throw `InvalidInputError` for unrecoverable input issues; for recoverable failures return `new ActionResult({ error, includeInMemory: true })` so the LLM can react.
- Wrap any string that originated from the page DOM in `wrapUntrustedContent()` from [`messages/utils`](chrome-extension/src/background/agent/messages/utils.ts) before placing it into `extractedContent`.
- Set `isDone: true` only for the terminal `done` action.

## 3. Registration

Construct the `Action` inside `ActionBuilder.buildDefaultActions()` and add it to the returned array. The Navigator picks them up via `NavigatorActionRegistry` during executor construction — there is no separate registry file to edit.

## Events & Logging

- Use `createLogger('Action')` already present in the file; do not introduce ad-hoc `console.log`.
- For long-running or user-visible steps, emit an `ExecutionState` event through the context's `EventManager` (see how existing actions call `context.emitEvent(Actors.NAVIGATOR, ExecutionState.ACT_*, ...)` patterns).

## Don'ts

- Don't access `chrome.*` APIs directly from a handler — go through `AgentContext.browserContext` / `BrowserPage`.
- Don't introduce a new action without a matching `act_<name>_{start,ok,fail}` i18n key set.
- Don't return `ActionResult` with `includeInMemory: false` for content the LLM needs to plan the next step.
