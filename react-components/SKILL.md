---
name: react-components
description: Write React components that are semantic, composed and markup-first — semantic HTML elements, screens as short compositions of plainly named components, state and effects in thin hooks that store only facts and derive the rest, and business logic as pure, unit-tested functions outside React. Use when writing, reviewing or refactoring any React component, hook or screen, when JSX is a soup of divs and class strings, when a component has a long run of state and effects before its return, or when logic is tangled into components.
---

# React components: semantic, composed, markup-first

A screen must be readable as what it is made of, by a person and by a screen reader. Apply these rules when writing or changing React components, in any project.

1. **Elements say what they are.** Use `main`, `header`, `footer`, `nav`, `section`, `search`, `ul`/`li`, `button`, `time`, `form`. A `div` or `span` is only for a box that exists for layout or scrolling.
   ```tsx
   // Not this: every box is a div, so a screen reader hears one long run of text, and
   // the "button" cannot be reached by Tab or pressed with Enter.
   <div className="screen">
     <div className="top">
       <span className="clock">10:42</span>
       <div className="search"><input placeholder="Search" /></div>
     </div>
     <div className="row">
       <div className="row-name">Explore</div>
       <div className="cards">
         <div className="card" onClick={choose}>Brainstorm</div>
       </div>
     </div>
   </div>

   // This: each element says what it is, so landmarks, lists and buttons come for free.
   <main>
     <header>
       <time dateTime="10:42">10:42</time>
       <search><input type="search" aria-label="Search activities" /></search>
     </header>
     <section aria-labelledby="explore">
       <h2 id="explore">Explore</h2>
       <ul>
         <li><button type="button" onClick={choose}>Brainstorm</button></li>
       </ul>
     </section>
   </main>
   ```
   What each choice gives: `main` and `header` let a screen reader jump between parts of the page; `section` with its `h2` names the region; `ul`/`li` says "list, 5 items"; `button` is reachable by Tab and pressed by Enter and Space with no extra code; `time` and `search` say what their content is. Keep a `div` where nothing is meant, such as the box that scrolls a row sideways.
2. **A screen is a composition of named parts.** A screen's JSX reads like an outline of the screen in the product's own words: one named component for each part a person would point at. Each part is a small component that owns its semantic element and its classes, so the screen itself has no raw HTML tags and no class strings. Reading the screen tells you what is on it; opening a part tells you how it looks.
   ```tsx
   // Not this: the screen is a tree of tags and classes, and what each box is has to be worked out.
   <div className="flex h-screen flex-col gap-5 pt-7">
     <div className="flex items-center gap-3 pl-14">
       <span className="text-text-helper">CBM Meet App</span>
       <span className="font-semibold">{name}</span>
     </div>
     <div className="min-h-0 flex-1 overflow-y-auto">
       {groups.map((g) => (
         <div key={g.group} className="flex flex-col gap-3">
           <div className="text-[1.625rem] font-semibold">{g.group}</div>
           <div className="flex gap-6 overflow-x-auto">{/* cards */}</div>
         </div>
       ))}
     </div>
     <button className="absolute bottom-2 left-11 rounded-full" onClick={onBack}>←</button>
   </div>

   // This: the screen is its parts, named for what they are.
   <ChoiceLayout heading="Choose an activity">
     <ChoiceHeader>
       <AppName />
       <SessionName name={name} />
     </ChoiceHeader>
     <ActivityRows groups={groups} onChoose={choose} />
     <BackButton label="Back to the dashboard" onBack={onBack} />
   </ChoiceLayout>

   // Each part holds its own element and classes, once.
   function ChoiceHeader({ children }: { children: ReactNode }) {
     return <header className="flex items-center gap-3 pl-14">{children}</header>;
   }
   ```
   A part is worth naming when a person would call it something ("the header", "a row of cards", "the back button"). A wrapper that exists only for spacing stays inside the part that needs it, not in the screen.
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
