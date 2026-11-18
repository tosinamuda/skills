---
name: react-components
description: Write React components that are semantic, composed and markup-first — semantic HTML elements, screens as short compositions of plainly named components, state and effects in thin hooks that store only facts and derive the rest, and business logic as pure, unit-tested functions outside React. Use when writing, reviewing or refactoring any React component, hook or screen, when JSX is a soup of divs and class strings, when a component has a long run of state and effects before its return, or when logic is tangled into components.
---

# React components: semantic, composed, markup-first

A screen must be readable as what it is made of, by a person and by a screen reader. Apply these rules when writing or changing React components, in any project.

1. **Elements say what they are.** Use `main`, `header`, `footer`, `nav`, `section`, `search`, `ul`/`li`, `button`, `time`, `form`. A `div` or `span` is only for a box that exists for layout or scrolling.
2. **A screen is a composition.** Its JSX is a short list of named components. A screen has no class strings; each part owns its own.
3. **Names a second-year student would guess.** Use the plain word for the thing: `Layout`, `Header`, `Row`, `Card`, `List`, `Panel`, `Button`, `Margin`. No metaphor and no trade jargon (`Frame`, `Rail`, `Strip`, `Gutter`, `Chrome`, `Shell`) unless the product's own users say that word. Test: someone who has never seen the code can say what `<ActivityRow>` draws.
4. **Markup first.** A component's body is its JSX. More than a few lines of state, effects or derivation before the `return` go into one hook named for what it models (`useActivityChoice`), which returns facts and actions.
5. **One tagged state for exclusive views.** When a screen shows exactly one of several things, a pure function or a hook returns a discriminated union that carries each state's data, and the screen is an exhaustive `switch` over it. No nested ternaries and no chain of early returns.
6. **Business logic lives outside React.** Anything that decides, checks, filters, sorts or computes is a pure function in a plain module that does not import React, so it can be reused and unit-tested without rendering. A hook or component calls it; it does not contain it.
7. **Children report, parents decide.** Props are events with facts (`onChoose(activity)`), never a parent's setter.
8. **Repeated markup becomes a component the second time it is written.**
9. **Hooks are thin and derive, not store.** A hook keeps only the state that cannot be computed: what the user did and what arrived from outside. Everything else is derived during render from that state, by the pure functions of rule 6. No state that mirrors a prop or another state, and no effect that only sets state from other state. An effect is for syncing with something outside React (a fetch, a timer, a subscription, the DOM), and cleans up after itself. One hook models one thing.
   ```ts
   // Store only what the user did and what arrived from outside.
   const [query, setQuery] = useState("");
   const { items } = useCatalog();                 // fetched; its effect cleans up

   // Derive the rest every render, with pure functions from a plain module.
   const shown = itemsMatching(items, query);
   const empty = shown.length === 0;

   // Not this: a copy that drifts, set by an effect that renders twice.
   // const [shown, setShown] = useState(items);
   // useEffect(() => { setShown(itemsMatching(items, query)); }, [items, query]);
   ```
10. **A split must not change what is drawn.** Record the rendered markup before and after and compare; a passing test suite does not show a moved wrapper.

Do not split for its own sake: a screen must still be readable whole, and a component with no rules needs no hook.
