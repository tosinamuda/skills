---
name: business-logic
description: Decide where a front-end app's business logic lives and how it is layered — pure domain rules, application flows that return outcomes, adapters at the edge, presentation models beside the UI — and how to pull logic that is woven into React components and hooks out into pure, unit-tested functions. Use when a component or hook decides, filters, computes or orchestrates calls, when logic can only be tested by rendering, when choosing folders for a feature's rules, or when the same rule is written in two places.
---

# Business logic: outside the framework, in layers, tested on its own

Business logic is any code that decides what is true or what happens next: a rule, a check, a calculation, a sequence of calls. Kept inside components and hooks, it can only be tested by rendering, only be reused by rendering, and only be read alongside markup. These rules keep it out.

## Where code belongs: the command-line test

Ask of each piece of code: **would it survive unchanged if the product shipped as a command-line tool?**

- **Yes** → it is domain or application code. It must not import React or any UI framework.
- **No, it decides what a screen shows** → it is a presentation model. Still pure and tested, but it lives with the UI.
- **No, it wires events to markup** → it is UI glue, and stays thin.

## The four layers

| Layer | What goes there | Imports |
| --- | --- | --- |
| **Domain** | Types and pure rules: what is true. "Is this basket over the free-delivery threshold?", "Which products match this search?" | Nothing outside itself |
| **Application** | Use cases: a sequence of steps that does something. "Place an order." It talks to the outside world through an interface it defines, and returns an outcome. | Domain |
| **Infrastructure** | Adapters that implement those interfaces: the HTTP client, storage, the clock. It maps the server's shapes to the app's own types. | Application, domain |
| **UI** | Components, hooks, and presentation models (what a screen shows). | All of the above |

Dependencies point inward: UI → application → domain, and infrastructure implements application's interfaces. A rename on the server changes one adapter, not every screen.

```text
src/features/orders/
├── domain/           order.ts, pricing.ts, pricing.test.ts
├── application/      placeOrder.ts, placeOrder.test.ts, OrderGateway.ts
├── infrastructure/   httpOrderGateway.ts
└── ui/               CheckoutScreen.tsx, useCheckout.ts, model/checkoutView.ts
```

A small feature does not need all four folders. Start with `domain/` (or `model/`) beside the UI, and add the others when a use case or a second data source appears.

## 1. Domain rules are pure functions

A rule takes everything it needs as arguments and returns a value. It does not read state, set state, fetch or log.

```ts
// domain/pricing.ts
export const FREE_DELIVERY_FROM = 50;

export function deliveryCost(subtotal: number): number {
  return subtotal >= FREE_DELIVERY_FROM ? 0 : 4.95;
}

// domain/pricing.test.ts
it("charges no delivery from fifty in the basket", () => {
  expect(deliveryCost(50)).toBe(0);
  expect(deliveryCost(49.99)).toBe(4.95);
});
```

## 2. Use cases return outcomes; screens decide what they mean

A use case does the work and reports what happened as a value. It never navigates, shows a toast or closes a dialog: those are the screen's words.

```ts
// Not this: the use case knows the screen's vocabulary.
async function placeOrder(basket: Basket, ui: { goToReceipt(): void; showError(m: string): void }) { ... }

// This: it returns an outcome a test can name, and any screen can act on.
export type PlaceOrderOutcome =
  | { kind: "placed"; orderId: string }
  | { kind: "outOfStock"; productIds: string[] }
  | { kind: "refused"; message: string };

export async function placeOrder(gateway: OrderGateway, basket: Basket): Promise<PlaceOrderOutcome> { ... }

// The screen maps the outcome to what happens on screen.
const outcome = await placeOrder(gateway, basket);
if (outcome.kind === "placed") navigate(`/orders/${outcome.orderId}`);
```

## 3. The outside world comes in through an interface

The use case defines the interface it needs. Production code passes the real adapter; a test passes a plain object.

```ts
// application/OrderGateway.ts
export interface OrderGateway {
  submit(basket: Basket): Promise<{ id: string } | { unavailable: string[] }>;
}

// application/placeOrder.test.ts
const gateway: OrderGateway = { submit: async () => ({ unavailable: ["p2"] }) };
expect(await placeOrder(gateway, basket)).toEqual({ kind: "outOfStock", productIds: ["p2"] });
```

Only add an interface where there is something to swap: a server, storage, a clock. Do not wrap pure functions in one.

## 4. Presentation models are pure too

What a screen shows is decided by a pure function returning a tagged state, kept with the UI and tested like a domain rule.

```ts
// ui/model/checkoutView.ts
export type CheckoutView = { kind: "empty" } | { kind: "ready"; total: number } | { kind: "tooSmall"; missing: number };

export function checkoutView(basket: Basket, minimum: number): CheckoutView {
  if (basket.items.length === 0) return { kind: "empty" };
  const total = subtotal(basket);
  return total < minimum ? { kind: "tooSmall", missing: minimum - total } : { kind: "ready", total };
}
```

## 5. Work that outlives a screen lives in a store

Polling, a background job, or state that several screens read belongs in a plain class, adapted to React with `useSyncExternalStore`. The class is unit-tested; the hook stays a few lines.

```ts
export class UploadStore {
  subscribe = (listener: () => void) => { this.listeners.add(listener); return () => this.listeners.delete(listener); };
  snapshot = () => this.uploads;
  start(file: File) { /* owns the request and its AbortController */ }
}

export const useUploads = (store: UploadStore) => useSyncExternalStore(store.subscribe, store.snapshot);
```

State that belongs to one screen's lifetime can stay in that screen's hook.

## 6. Pulling logic out of a component

When a component or hook decides, filters, sorts or computes:

1. **Find the decision.** An `if` about data, a `.filter` or `.sort`, a calculation in `useMemo`, a check in an event handler.
2. **Make its inputs explicit.** Everything it reads from state, props or refs becomes a parameter.
3. **Return a value instead of setting state.** `setTooSmall(total < minimum)` becomes a function that returns the answer.
4. **Move it to a plain module named for its concern** (`pricing.ts`, `productSearch.ts`), never `utils.ts` or `helpers.ts`.
5. **Test it there, then break it on purpose.** Change the rule (`>=` to `>`) and confirm a named test fails. A test that still passes proves nothing.
6. **Replace the original code with one call.**

```tsx
// Before: the rule is inside the component, and testing it means rendering.
function BasketFooter({ items }: Props) {
  const subtotal = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  const delivery = subtotal >= 50 ? 0 : 4.95;
  return <footer>Delivery: {delivery === 0 ? "free" : delivery}</footer>;
}

// After: the component only calls the rules.
function BasketFooter({ items }: Props) {
  const delivery = deliveryCost(subtotal(items));
  return <footer>Delivery: {delivery === 0 ? "free" : delivery}</footer>;
}
```

## Where tests run

- Domain, application and presentation models run in a plain test environment with no DOM. **Needing a DOM to test a rule means the rule is still tangled with rendering.**
- Component tests check the wiring: the right part shows, and events reach the use case. The rules are already proved below them.

## When not to layer

A screen that only lists and edits records can call its gateway from a small hook and stop there. A layer earns its place with a rule, a second caller, or work that outlives the screen; never add one for a feature that has none.
