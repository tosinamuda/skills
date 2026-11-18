---
name: business-logic
description: Decide where a React app's code goes — components, hooks, api, utils, types and constants — so business rules are pure, unit-tested functions outside React, server calls live in plain api functions that React Query or a hook consumes, and components only draw. Covers how to pull logic woven into components and hooks out into tested functions. Use when a component or hook decides, filters, computes or makes server calls, when logic can only be tested by rendering, when setting up React Query, when choosing folders for a feature, or when the same rule is written in two places.
---

# Business logic: where each kind of code goes

Business logic is any code that decides what is true or what happens next: a rule, a check, a calculation. Kept inside components and hooks, it can only be tested by rendering and only be reused by rendering. These rules give each kind of code one home, so the rules are plain functions with their own tests.

## The folders of a feature

```text
src/features/orders/
├── components/    what is drawn: JSX and its classes, nothing else
├── hooks/         React glue: state, effects, React Query hooks
├── api/           calls to the server: plain async functions, and their query options
├── utils/         business rules: pure functions, each with a test beside it
├── types.ts       the feature's own types
└── constants.ts   named values the rules and screens share
```

| Folder | Holds | Imports React? | Tested by |
| --- | --- | --- | --- |
| `components/` | Markup. A component calls a hook and draws what it returns. | Yes | Rendering, for wiring only |
| `hooks/` | `useState`, effects, `useQuery`, `useMutation`. A hook calls `api/` and `utils/`; it holds no rules itself. | Yes | Rarely needed, since it holds no rules |
| `api/` | One function for each server call, mapping the server's shapes to the feature's types; the query keys and `queryOptions` for them. | No (`queryOptions` from React Query is a plain function) | A fake `fetch`, or not at all if it only passes data through |
| `utils/` | Rules and calculations: totals, checks, filters, sorting, what a screen shows. | No | Plain unit tests, no DOM |
| `types.ts` | Types the feature works with: `Order`, `Basket`, `Product`. | No | — |
| `constants.ts` | Values with a name and a reason: `FREE_DELIVERY_FROM = 50`. | No | — |

Dependencies point one way: `components` → `hooks` → `api` and `utils` → `types` and `constants`. `utils/` never imports `api/` or `hooks/`; `api/` never imports `hooks/` or `components/`.

The folder names follow Bulletproof React, so they are familiar to most React developers. Two places differ from it on purpose. Bulletproof describes `utils/` only as "utility functions"; here it is where the feature's business rules go, so each file is named for its concern (`pricing.ts`, `productSearch.ts`); there is never a `utils.ts` or `helpers.ts` file that collects unrelated functions. State that outlives a screen, such as an upload several screens watch, goes in a `stores/` folder beside them. And Bulletproof puts the React Query hooks in `api/` with the requests; here `api/` stays free of React and the hooks go in `hooks/`, so the requests can be called and tested without a component.

A small feature does not need every folder. A screen that only lists records can be a component, one hook and one api function. Add `utils/` the first time a rule appears.

## 1. `utils/` holds the rules, as pure functions

A rule takes everything it needs as arguments and returns a value. It does not read or set state, fetch, or log.

```ts
// constants.ts
/** Orders from this amount ship free. */
export const FREE_DELIVERY_FROM = 50;
export const DELIVERY_COST = 4.95;

// utils/pricing.ts
import { DELIVERY_COST, FREE_DELIVERY_FROM } from "../constants";
import type { BasketItem } from "../types";

export const subtotal = (items: readonly BasketItem[]) => items.reduce((sum, i) => sum + i.price * i.quantity, 0);
export const deliveryCost = (subtotal: number) => (subtotal >= FREE_DELIVERY_FROM ? 0 : DELIVERY_COST);

// utils/pricing.test.ts
it("charges no delivery from the free-delivery amount", () => {
  expect(deliveryCost(50)).toBe(0);
  expect(deliveryCost(49.99)).toBe(4.95);
});
```

What a screen shows is a rule too, and lives in `utils/` as a function returning a tagged state:

```ts
// utils/checkoutView.ts
export type CheckoutView = { kind: "empty" } | { kind: "tooSmall"; missing: number } | { kind: "ready"; total: number };

export function checkoutView(items: readonly BasketItem[], minimum: number): CheckoutView {
  if (items.length === 0) return { kind: "empty" };
  const total = subtotal(items);
  return total < minimum ? { kind: "tooSmall", missing: minimum - total } : { kind: "ready", total };
}
```

## 2. `api/` holds plain server calls; hooks use them through React Query

Each server call is an async function with no React in it. It maps the server's response to the feature's own types, so a renamed field on the server changes one file. The query key and the `queryOptions` sit beside it, so every hook and prefetch uses the same key.

```ts
// api/products.ts
import { queryOptions } from "@tanstack/react-query";
import type { Product } from "../types";

interface ProductResponse { product_id: string; display_name: string; unit_price_cents: number }

const toProduct = (r: ProductResponse): Product => ({ id: r.product_id, name: r.display_name, price: r.unit_price_cents / 100 });

export async function fetchProducts(category: string): Promise<Product[]> {
  const response = await fetch(`/api/products?category=${encodeURIComponent(category)}`, { signal: AbortSignal.timeout(10_000) });
  if (!response.ok) throw new Error(`The products could not be loaded (${response.status})`);
  return ((await response.json()) as ProductResponse[]).map(toProduct);
}

export const productKeys = {
  all: ["products"] as const,
  byCategory: (category: string) => ["products", category] as const,
};

export const productsOptions = (category: string) =>
  queryOptions({ queryKey: productKeys.byCategory(category), queryFn: () => fetchProducts(category) });
```

```ts
// hooks/useProductSearch.ts
export function useProductSearch(category: string) {
  const [query, setQuery] = useState("");                         // what the user typed
  const products = useQuery(productsOptions(category));            // what the server sent; React Query owns it
  const shown = productsMatching(products.data ?? [], query);      // derived, by a rule in utils/
  return { query, setQuery, shown, loading: products.isPending, error: products.error };
}
```

Server data lives in React Query's cache. Never copy it into `useState`: that makes a second copy that goes stale.

A change on the server is a mutation. The api function does the call; the hook refreshes what it changed:

```ts
// api/orders.ts
export async function submitOrder(items: readonly BasketItem[]): Promise<OrderResult> { ... }

// hooks/usePlaceOrder.ts
export function usePlaceOrder() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: submitOrder,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: orderKeys.all }),
  });
}
```

## 3. Calls that do several steps return an outcome, not a screen action

When placing an order means checking stock, then submitting, the steps go in one `api/` function that reports what happened as a value. It never navigates, shows a message or closes a dialog; the component decides that.

```ts
// types.ts
export type PlaceOrderOutcome =
  | { kind: "placed"; orderId: string }
  | { kind: "outOfStock"; productIds: string[] };

// api/orders.ts
export async function placeOrder(items: readonly BasketItem[]): Promise<PlaceOrderOutcome> {
  const missing = await checkStock(items);
  if (missing.length > 0) return { kind: "outOfStock", productIds: missing };
  const { id } = await submitOrder(items);
  return { kind: "placed", orderId: id };
}

// components/CheckoutButton.tsx: the screen maps the outcome to what happens on screen.
const order = usePlaceOrder();
const outcome = await order.mutateAsync(items);
if (outcome.kind === "placed") navigate(`/orders/${outcome.orderId}`);
```

## 4. `types.ts` and `constants.ts`

- **`types.ts`** holds the types the feature works with, in its own words. The server's response shapes stay private to `api/`, which maps them to these.
- **`constants.ts`** holds values that have a name and a reason, used in more than one place. A value used by one rule stays in that rule's file. Give each one a comment saying why it is that value.
- Types or constants shared by several features move to `src/types.ts` and `src/constants.ts`, the second time they are needed.

## 5. Pulling logic out of a component

When a component or hook decides, filters, sorts or computes:

1. **Find the decision.** An `if` about data, a `.filter` or `.sort`, a calculation in `useMemo`, a check in an event handler.
2. **Make its inputs explicit.** Everything it reads from state, props or refs becomes a parameter.
3. **Return a value instead of setting state.** `setTooSmall(total < minimum)` becomes a function that returns the answer.
4. **Move it to `utils/`, in a file named for its concern** (`pricing.ts`, `productSearch.ts`), never `utils.ts` or `helpers.ts`. A number in it with a meaning moves to `constants.ts`.
5. **Test it there, then break it on purpose.** Change the rule (`>=` to `>`) and confirm a named test fails. A test that still passes proves nothing.
6. **Replace the original code with one call.**

```tsx
// Before: the rule and its numbers are inside the component, and testing them means rendering.
function BasketFooter({ items }: Props) {
  const subtotal = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  const delivery = subtotal >= 50 ? 0 : 4.95;
  return <footer>Delivery: {delivery === 0 ? "free" : delivery}</footer>;
}

// After: the component draws, and the rules are tested in utils/.
function BasketFooter({ items }: Props) {
  const delivery = deliveryCost(subtotal(items));
  return <footer>Delivery: {delivery === 0 ? "free" : delivery}</footer>;
}
```

## Where tests run

- `utils/` runs in a plain test environment with no DOM. **Needing a DOM to test a rule means the rule is still tangled with rendering.**
- `api/` is tested with a fake `fetch` when it maps or decides something; a function that only passes data through needs no test of its own.
- Component tests check the wiring: the right part shows, and events reach the hook. The rules are already proved in `utils/`.
