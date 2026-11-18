---
name: business-logic
description: Organise a React feature the Bulletproof React way — api, components, hooks, stores, types, utils — with its business rules in a separate services folder as pure, unit-tested functions outside React, so components only draw and hooks only connect. Covers React Query in the api folder, where constants and types go, and how to pull logic woven into components and hooks out into tested functions. Use when a component or hook decides, filters, computes or orchestrates server calls, when logic can only be tested by rendering, when setting up React Query, when choosing folders for a feature, or when the same rule is written in two places.
---

# Business logic: a feature in Bulletproof React's folders, with its rules in `services/`

Business logic is any code that decides what is true or what happens next: a rule, a check, a calculation, a sequence of server calls. Kept inside components and hooks, it can only be tested by rendering and only be reused by rendering. These rules give it one home of its own, `services/`, next to the folders Bulletproof React already uses.

## The folders of a feature

```text
src/features/orders/
├── api/          one file per server call: its types, its request function and its React Query hook
├── components/   what is drawn: JSX and its classes
├── hooks/        hooks that are not server calls: state, effects, browser APIs
├── services/     business rules and flows: plain functions with no React, each with a test beside it
├── stores/       state several screens share or that outlives a screen
├── types/        the feature's own types
└── utils/        small generic helpers: formatting, dates, parsing
```

| Folder | Holds | Imports React? | Tested by |
| --- | --- | --- | --- |
| `api/` | Per endpoint, as Bulletproof React does: request and response types, a request function using the app's API client, and a `useQuery` or `useMutation` hook around it. | The hook does; the request function does not | The request function, with a fake client, when it maps or checks something |
| `components/` | Markup. A component calls hooks and draws what they return. | Yes | Rendering, for wiring only |
| `hooks/` | Hooks that are not server calls: what the user typed, which item is open, a timer, the window size. A hook calls `services/` for any rule. | Yes | Rarely needed, since it holds no rules |
| `services/` | The business rules: prices, checks, filters, sorting, what a screen shows, and flows of several server calls that return an outcome. | **Never** | Plain unit tests, no DOM |
| `stores/` | Shared or long-lived state, in the app's store library. | Through the store library | Its actions, as plain functions |
| `types/` | Types the feature works with, in its own words. | No | — |
| `utils/` | Helpers with no business meaning that any feature could use: formatting a price, parsing a date. | No | Unit tests |

The line between `services/` and `utils/`: a function that would change if the business changed its mind (a delivery threshold, who may cancel an order) is a service. A function that would stay the same in any business (format a number as money) is a util.

Dependencies point one way: `components` → `hooks` and `api` → `services` → `types`. `services/` never imports `api/` hooks, `hooks/`, `stores/` or `components/`. Shared code across features lives in `src/components`, `src/hooks`, `src/lib`, `src/types`, `src/utils` and `src/config`, as in Bulletproof React.

A small feature does not need every folder. A screen that only lists records can be a component and one `api/` file. Add `services/` the first time a rule appears.

## 1. Rules in `services/` are pure functions

A rule takes everything it needs as arguments and returns a value. It does not read or set state, fetch, or log. A value with business meaning is a named constant beside the rule that uses it, with a comment saying why it is that value.

```ts
// services/pricing.ts
import type { BasketItem } from "../types";

/** Orders from this amount ship free: the threshold marketing promises. */
export const FREE_DELIVERY_FROM = 50;
export const DELIVERY_COST = 4.95;

export const subtotal = (items: readonly BasketItem[]) => items.reduce((sum, i) => sum + i.price * i.quantity, 0);
export const deliveryCost = (subtotal: number) => (subtotal >= FREE_DELIVERY_FROM ? 0 : DELIVERY_COST);

// services/pricing.test.ts
it("charges no delivery from the free-delivery amount", () => {
  expect(deliveryCost(50)).toBe(0);
  expect(deliveryCost(49.99)).toBe(4.95);
});
```

A constant used by several services goes in `services/constants.ts`. Configuration that is not a business rule (an API base URL, a feature flag) goes in `src/config`.

What a screen shows is a rule too, and lives in `services/` as a function returning a tagged state:

```ts
// services/checkout-view.ts
export type CheckoutView = { kind: "empty" } | { kind: "tooSmall"; missing: number } | { kind: "ready"; total: number };

export function checkoutView(items: readonly BasketItem[], minimum: number): CheckoutView {
  if (items.length === 0) return { kind: "empty" };
  const total = subtotal(items);
  return total < minimum ? { kind: "tooSmall", missing: minimum - total } : { kind: "ready", total };
}
```

## 2. `api/` follows Bulletproof React: one file per endpoint

Each file holds the endpoint's types, a request function using the app's API client, and the React Query hook that wraps it. The request function maps the server's shape to the feature's own type, so a renamed field on the server changes one file.

```ts
// api/get-products.ts
import { queryOptions, useQuery } from "@tanstack/react-query";
import { api } from "@/lib/api-client";
import type { Product } from "../types";

type ProductResponse = { product_id: string; display_name: string; unit_price_cents: number };

const toProduct = (r: ProductResponse): Product => ({ id: r.product_id, name: r.display_name, price: r.unit_price_cents / 100 });

export const getProducts = async (category: string): Promise<Product[]> =>
  (await api.get<ProductResponse[]>("/products", { params: { category } })).map(toProduct);

export const getProductsQueryOptions = (category: string) =>
  queryOptions({ queryKey: ["products", category], queryFn: () => getProducts(category) });

export const useProducts = (category: string) => useQuery(getProductsQueryOptions(category));
```

A hook that combines server data with what the user did calls `services/` for the rule, and stores only what it must:

```ts
// hooks/use-product-search.ts
export function useProductSearch(category: string) {
  const [query, setQuery] = useState("");                       // what the user typed
  const products = useProducts(category);                        // what the server sent; React Query owns it
  const shown = productsMatching(products.data ?? [], query);    // derived, by a rule in services/
  return { query, setQuery, shown, loading: products.isPending, error: products.error };
}
```

Server data lives in React Query's cache. Never copy it into `useState`: that makes a second copy that goes stale.

## 3. A flow of several calls is a service that returns an outcome

When placing an order means checking stock and then submitting, the sequence is a service. It takes the request functions it needs as arguments, so a test passes fakes, and it reports what happened as a value. It never navigates, shows a message or closes a dialog; the component decides that.

```ts
// services/place-order.ts
export type PlaceOrderOutcome =
  | { kind: "placed"; orderId: string }
  | { kind: "outOfStock"; productIds: string[] };

type Requests = {
  checkStock: (items: readonly BasketItem[]) => Promise<string[]>;
  submitOrder: (items: readonly BasketItem[]) => Promise<{ id: string }>;
};

export async function placeOrder(items: readonly BasketItem[], requests: Requests): Promise<PlaceOrderOutcome> {
  const missing = await requests.checkStock(items);
  if (missing.length > 0) return { kind: "outOfStock", productIds: missing };
  const { id } = await requests.submitOrder(items);
  return { kind: "placed", orderId: id };
}

// services/place-order.test.ts
it("does not submit an order with products out of stock", async () => {
  const submitOrder = vi.fn();
  const outcome = await placeOrder(basket, { checkStock: async () => ["p2"], submitOrder });
  expect(outcome).toEqual({ kind: "outOfStock", productIds: ["p2"] });
  expect(submitOrder).not.toHaveBeenCalled();
});
```

```ts
// api/place-order.ts: the mutation hook runs the service with the real requests.
export const usePlaceOrder = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (items: readonly BasketItem[]) => placeOrder(items, { checkStock, submitOrder }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ["orders"] }),
  });
};

// components/checkout-button.tsx: the screen maps the outcome to what happens on screen.
const outcome = await placeOrderMutation.mutateAsync(items);
if (outcome.kind === "placed") navigate(`/orders/${outcome.orderId}`);
```

## 4. Pulling logic out of a component

When a component or hook decides, filters, sorts or computes:

1. **Find the decision.** An `if` about data, a `.filter` or `.sort`, a calculation in `useMemo`, a check in an event handler.
2. **Make its inputs explicit.** Everything it reads from state, props or refs becomes a parameter.
3. **Return a value instead of setting state.** `setTooSmall(total < minimum)` becomes a function that returns the answer.
4. **Move it to `services/`, in a file named for its concern** (`pricing.ts`, `product-search.ts`). A number in it with a business meaning becomes a named constant with a comment.
5. **Test it there, then break it on purpose.** Change the rule (`>=` to `>`) and confirm a named test fails. A test that still passes proves nothing.
6. **Replace the original code with one call.**

```tsx
// Before: the rule and its numbers are inside the component, and testing them means rendering.
function BasketFooter({ items }: Props) {
  const subtotal = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  const delivery = subtotal >= 50 ? 0 : 4.95;
  return <footer>Delivery: {delivery === 0 ? "free" : formatPrice(delivery)}</footer>;
}

// After: the component draws; the rules are tested in services/, and formatPrice is a util.
function BasketFooter({ items }: Props) {
  const delivery = deliveryCost(subtotal(items));
  return <footer>Delivery: {delivery === 0 ? "free" : formatPrice(delivery)}</footer>;
}
```

## Where tests run

- `services/` and `utils/` run in a plain test environment with no DOM. **Needing a DOM to test a rule means the rule is still tangled with rendering.**
- `api/` request functions are tested with a fake client when they map or check something; one that only passes data through needs no test of its own.
- Component tests check the wiring: the right part shows, and events reach the hook. The rules are already proved in `services/`.
