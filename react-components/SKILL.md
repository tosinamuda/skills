---
name: react-components
description: Write React components that are semantic, composed and markup-first — semantic HTML elements, screens as short compositions of plainly named components, state and effects in thin hooks that store only facts and derive the rest, and business logic kept outside React (see the business-logic skill). Use when writing, reviewing or refactoring any React component, hook or screen, when JSX is a soup of divs and class strings, when a component has a long run of state and effects before its return, or when logic is tangled into components.
---

# React components: semantic, composed, markup-first

A screen must be readable as what it is made of, by a person and by a screen reader. Apply these rules when writing or changing React components, in any project.

1. **Elements say what they are.** Use `main`, `header`, `footer`, `nav`, `section`, `search`, `ul`/`li`, `button`, `time`, `form`. A `div` or `span` is only for a box that exists for layout or scrolling.
   ```tsx
   // Not this: every box is a div, so a screen reader hears one long run of text, and
   // the "button" cannot be reached by Tab or pressed with Enter.
   <div className="page">
     <div className="top">
       <span className="updated">10:42</span>
       <div className="search"><input placeholder="Search" /></div>
     </div>
     <div className="group">
       <div className="group-name">Fruit</div>
       <div className="items">
         <div className="item" onClick={add}>Apples</div>
       </div>
     </div>
   </div>

   // This: each element says what it is, so landmarks, lists and buttons come for free.
   <main>
     <header>
       <time dateTime="10:42">10:42</time>
       <search><input type="search" aria-label="Search products" /></search>
     </header>
     <section aria-labelledby="fruit">
       <h2 id="fruit">Fruit</h2>
       <ul>
         <li><button type="button" onClick={add}>Apples</button></li>
       </ul>
     </section>
   </main>
   ```
   What each choice gives: `main` and `header` let a screen reader jump between parts of the page; `section` with its `h2` names the region; `ul`/`li` says "list, 5 items"; `button` is reachable by Tab and pressed by Enter and Space with no extra code; `time` and `search` say what their content is. Keep a `div` where nothing is meant, such as the box that scrolls a list sideways.
2. **A screen is a composition of named parts.** A screen's JSX reads like an outline of the screen in the product's own words: one named component for each part a person would point at. Each part is a small component that owns its semantic element and its classes, so the screen itself has no raw HTML tags and no class strings. Reading the screen tells you what is on it; opening a part tells you how it looks.
   ```tsx
   // Not this: the screen is a tree of tags and classes, and what each box is has to be worked out.
   <div className="flex h-screen flex-col gap-5 pt-7">
     <div className="flex items-center gap-3 px-8">
       <span className="text-gray-500">Shop</span>
       <span className="font-semibold">{storeName}</span>
     </div>
     <div className="min-h-0 flex-1 overflow-y-auto">
       {categories.map((c) => (
         <div key={c.name} className="flex flex-col gap-3">
           <div className="text-2xl font-semibold">{c.name}</div>
           <div className="flex gap-6 overflow-x-auto">{/* product cards */}</div>
         </div>
       ))}
     </div>
     <button className="absolute bottom-2 left-8 rounded-full" onClick={onBack}>←</button>
   </div>

   // This: the screen is its parts, named for what they are.
   <CatalogLayout heading="Products">
     <CatalogHeader>
       <ShopName />
       <StoreName name={storeName} />
     </CatalogHeader>
     <CategoryList categories={categories} onAdd={add} />
     <BackButton label="Back to the store" onBack={onBack} />
   </CatalogLayout>

   // Each part holds its own element and classes, once.
   function CatalogHeader({ children }: { children: ReactNode }) {
     return <header className="flex items-center gap-3 px-8">{children}</header>;
   }
   ```
   A part is worth naming when a person would call it something ("the header", "a row of products", "the back button"). A wrapper that exists only for spacing stays inside the part that needs it, not in the screen.
3. **Names a second-year student would guess.** Use the plain word for the thing: `Layout`, `Header`, `Row`, `Card`, `List`, `Panel`, `Button`, `Margin`. No metaphor and no trade jargon (`Frame`, `Rail`, `Strip`, `Gutter`, `Chrome`, `Shell`) unless the product's own users say that word. Test: someone who has never seen the code can say what `<ProductRow>` draws.
   ```tsx
   // Not this: metaphor and jargon a newcomer has to decode.
   <PageFrame><TopRail /><ProductStrip gutter={GUTTER} /></PageFrame>

   // This: the plain word for each thing.
   <PageLayout><PageHeader /><ProductRow margin={LEFT_MARGIN} /></PageLayout>
   ```
4. **Markup first.** A component's body is its JSX. More than a few lines of state, effects or derivation before the `return` go into one hook named for what it models (`useProductSearch`), which returns facts and actions.
   ```tsx
   // Not this: forty lines of state and effects before anyone can see what is drawn.
   function CatalogScreen({ onAdd }: Props) {
     const [products, setProducts] = useState<Product[] | null>(null);
     const [query, setQuery] = useState("");
     const [adding, setAdding] = useState<string | null>(null);
     useEffect(() => { /* fetch the products, guard against unmount */ }, []);
     const categories = useMemo(() => /* filter, then group */, [products, query]);
     const add = (p: Product) => { /* set adding, call onAdd, catch */ };
     // ... thirty more lines ...
     return <CatalogLayout>...</CatalogLayout>;
   }

   // This: one hook named for what it models, then the JSX.
   function CatalogScreen({ onAdd }: Props) {
     const search = useProductSearch({ onAdd });
     return (
       <CatalogLayout>
         <CategoryList categories={search.categories} onAdd={search.add} />
       </CatalogLayout>
     );
   }
   ```
5. **One tagged state for exclusive views.** When a screen shows exactly one of several things, a pure function or a hook returns a discriminated union that carries each state's data, and the screen is an exhaustive `switch` over it. No nested ternaries and no chain of early returns.
   ```tsx
   // Not this: a chain of conditions, where the order is the logic and a case is easy to miss.
   if (!loaded) return <Spinner />;
   if ("error" in loaded) return <Notice title="Could not load the order" />;
   if (order.cancelled) return <Notice title="Cancelled" />;
   return order.items.length === 0 ? <EmptyBasket /> : order.paid ? <Receipt /> : <Checkout />;

   // This: one function decides, and returns the data each case needs.
   type OrderView =
     | { kind: "loading" }
     | { kind: "failed"; error: string }
     | { kind: "cancelled" }
     | { kind: "empty" }
     | { kind: "checkout"; order: Order }
     | { kind: "paid"; order: Order; receipt: Receipt };

   const view = useOrderView(id);
   switch (view.kind) {
     case "loading":   return <Spinner />;
     case "failed":    return <Notice title="Could not load the order" detail={view.error} />;
     case "cancelled": return <Notice title="Cancelled" />;
     case "empty":     return <EmptyBasket />;
     case "checkout":  return <Checkout order={view.order} />;
     case "paid":      return <ReceiptPage order={view.order} receipt={view.receipt} />;
   }
   ```
   With `kind` exhaustive, TypeScript fails the build when a new case is added and not drawn.
6. **Business logic lives outside React.** Anything that decides, checks, filters, sorts or computes is a pure function in a plain module that does not import React, so it can be reused and unit-tested without rendering. A hook or component calls it; it does not contain it. Where that code goes, how to layer it and how to get it out of a component are in the **business-logic** skill.
   ```ts
   // productSearch.ts: no React import, so it runs and is tested on its own.
   export function productsMatching(products: readonly Product[], query: string) {
     const words = query.toLowerCase().split(/\s+/).filter(Boolean);
     return products.filter((p) => words.every((w) => p.name.toLowerCase().includes(w)));
   }

   // The hook only calls it.
   const shown = productsMatching(products, query);
   ```
7. **Children report, parents decide.** Props are events with facts (`onAdd(product)`), never a parent's setter.
   ```tsx
   // Not this: the child is handed the parent's setters and decides what adding means.
   <ProductCard product={p} setBasket={setBasket} setError={setError} />

   // This: the child reports what happened, with the facts; the parent decides.
   <ProductCard product={p} onAdd={() => { add(p); }} />
   ```
8. **Repeated markup becomes a component the second time it is written.**
   ```tsx
   // Not this: the same page background and padding written out in four screens.
   <div className="relative flex h-screen w-screen flex-col bg-gray-50 p-8">...</div>

   // This: written once, the second time it was needed.
   <PageLayout>...</PageLayout>
   ```
9. **Hooks are thin and derive, not store.** A hook keeps only the state that cannot be computed: what the user did and what arrived from outside. Everything else is derived during render from that state, by the pure functions of rule 6. No state that mirrors a prop or another state, and no effect that only sets state from other state. An effect is for syncing with something outside React (a fetch, a timer, a subscription, the DOM), and cleans up after itself. One hook models one thing.
   ```ts
   // Store only what the user did and what arrived from outside.
   const [query, setQuery] = useState("");
   const { products } = useProducts();                 // fetched; its effect cleans up

   // Derive the rest every render, with pure functions from a plain module.
   const shown = productsMatching(products, query);
   const empty = shown.length === 0;

   // Not this: a copy that drifts, set by an effect that renders twice.
   // const [shown, setShown] = useState(products);
   // useEffect(() => { setShown(productsMatching(products, query)); }, [products, query]);
   ```
10. **A split must not change what is drawn.** Record the rendered markup before and after and compare; a passing test suite does not show a moved wrapper.
   ```bash
   # Record each screen's rendered markup before the change, make the change, record again, and compare.
   node record-markup.mjs before/   # your own Playwright script: opens each screen state, saves document.body as HTML
   node record-markup.mjs after/
   diff -r before/ after/           # only the elements you meant to change should differ
   ```
   Strip what changes from run to run (ids, times, generated codes) before comparing, so a real difference stands out.

Do not split for its own sake: a screen must still be readable whole, and a component with no rules needs no hook.
