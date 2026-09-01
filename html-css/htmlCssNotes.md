# HTML, CSS & Frontend Styling Frameworks — Deep Dive Roadmap

We'll go from fundamentals → layout systems → responsive design → CSS architecture → preprocessors → utility/component frameworks → performance & accessibility → production.

---

## 1. HTML Fundamentals

**Definition:** HTML (HyperText Markup Language) is the markup language that defines the **structure and content** of a web page — a tree of elements the browser parses into the DOM (Document Object Model), which CSS then styles and JavaScript then makes interactive. HTML is concerned with *what* content is and *what it means*, not how it looks (that's CSS's job) or how it behaves (that's JS's job).

**Document structure — Definition:** every HTML document starts with a `<!DOCTYPE html>` declaration (tells the browser to use standards-compliant rendering, not legacy "quirks mode"), followed by an `<html>` root element containing a `<head>` (metadata, not rendered directly) and a `<body>` (the actual visible content).

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>My Page</title>
</head>
<body>
  <h1>Hello</h1>
</body>
</html>
```

**Elements, tags, attributes — Definition:** an **element** is a piece of content marked up with a start and end **tag** (`<p>...</p>`) or a self-closing tag (`<img />`); **attributes** are key-value pairs inside the opening tag providing extra information about the element (`<img src="..." alt="...">`).

**Block vs inline elements — Definition:** a **block-level** element (`<div>`, `<p>`, `<h1>`) starts on a new line and takes up the full available width by default; an **inline** element (`<span>`, `<a>`, `<strong>`) flows within the surrounding text, only as wide as its content, and cannot contain block-level children. (`display`, covered in section 3, is what actually determines this — the "block vs inline" tag categorization is really just each element's browser-default `display` value.)

**Semantic HTML — Definition:** using elements whose *name* conveys the *meaning/role* of the content they wrap, rather than a generic `<div>`/`<span>` for everything — e.g. `<header>` (introductory content for a page or section), `<nav>` (navigation links), `<main>` (the page's primary content, one per page), `<section>` (a thematic grouping of content, typically with its own heading), `<article>` (independently distributable content, like a blog post), `<aside>` (tangentially related content, like a sidebar), `<footer>` (closing/metadata content for a page or section).

```html
<body>
  <header><nav>...</nav></header>
  <main>
    <article>
      <h2>Post Title</h2>
      <section>...</section>
    </article>
    <aside>Related links</aside>
  </main>
  <footer>© 2026</footer>
</body>
```

**Why semantics matter — Definition:** semantic elements give screen readers and other assistive technology a meaningful structure to navigate by (accessibility, section 13), give search engines stronger signals about content importance/structure (SEO), and make the markup itself more self-documenting for other developers — a `<div class="nav">` conveys nothing to a screen reader or search crawler the way `<nav>` does natively.

**Forms & form elements — Definition:** `<form>` wraps a set of interactive controls (`<input>`, `<select>`, `<textarea>`, `<button>`) for collecting user input, submitted to a server (or intercepted by JS) — see the React/Angular notes' forms sections for how frameworks layer state management on top of these native elements.

**Tables — Definition:** `<table>`/`<tr>`/`<td>`/`<th>` mark up genuinely **tabular data** (rows and columns of related values) — a common misuse in older web development was using tables for page *layout*, now considered an anti-pattern superseded by Flexbox/Grid (section 4); tables should be reserved for actual tabular data today.

**Media elements — Definition:** `<img>` (a single image), `<video>`/`<audio>` (media playback, with native browser controls), `<picture>` (lets the browser choose among multiple image sources based on viewport size/format support — the foundation of responsive images, section 5).

**Metadata — Definition:** `<meta charset="UTF-8">` declares the document's character encoding; `<meta name="viewport" content="width=device-width, initial-scale=1.0">` tells mobile browsers to render at the device's actual width rather than a desktop-assumed default (essential for any responsive design, section 5, to work at all on mobile).

**The DOM (brief) — Definition:** the Document Object Model is the browser's in-memory, tree-structured representation of the parsed HTML, which JavaScript can read and manipulate — the actual live "document" that CSS styles are applied to and that frameworks like React/Angular (see those notes) ultimately update.

---

## 2. CSS Fundamentals

**Definition:** CSS (Cascading Style Sheets) is the language that describes the **presentation** of HTML content — layout, color, typography, spacing — separating *how something looks* from *what it is* (HTML's job) and *how it behaves* (JavaScript's job).

**How CSS is applied — Definition:**
```html
<p style="color: red;">inline</p>                  <!-- inline: on the element itself, highest specificity -->
<style>p { color: red; }</style>                    <!-- internal: in a <style> block in the document -->
<link rel="stylesheet" href="styles.css" />          <!-- external: a separate .css file — the standard approach -->
```
External stylesheets are preferred for real projects: cacheable separately from the HTML, reusable across pages, and keep structure/presentation cleanly separated.

**Selectors — Definition:** patterns that determine which elements a CSS rule applies to.

```css
p { }                    /* element selector — all <p> tags */
.card { }                /* class selector — any element with class="card" */
#header { }               /* ID selector — the one element with id="header" */
.card p { }               /* descendant combinator — any <p> anywhere inside .card */
.card > p { }             /* child combinator — only <p> DIRECT children of .card */
.card + p { }             /* adjacent sibling — a <p> immediately following .card */
.card ~ p { }             /* general sibling — any <p> following .card at the same level */
input[type="email"] { }   /* attribute selector */
a:hover { }               /* pseudo-class — a state (hover, focus, first-child, ...) */
p::first-line { }         /* pseudo-element — targets a sub-part of an element's content */
```

**The Cascade — Definition:** the algorithm that determines which of potentially several conflicting CSS rules actually applies to a given element — resolved, in order, by: **origin & importance** (author styles vs browser defaults vs `!important`), then **specificity** (below), then **source order** (the later rule wins if specificity is tied) — "cascading" refers to this layered resolution process, not just that styles "flow down."

**Specificity — Definition:** a weight calculated for each selector, determining which of two competing rules wins when both target the same element, roughly: inline styles > ID selectors > class/attribute/pseudo-class selectors > element/pseudo-element selectors — a more specific selector always wins regardless of source order.

```css
p { color: blue; }              /* specificity: 0-0-1 (one element selector) */
.text { color: green; }          /* specificity: 0-1-0 (one class) — wins over the above */
#main .text { color: red; }      /* specificity: 1-1-0 (one ID + one class) — wins over both */
```

**Inheritance — Definition:** certain CSS properties (mostly text-related: `color`, `font-family`, `line-height`) automatically pass down from a parent element to its children unless explicitly overridden; other properties (mostly box/layout-related: `margin`, `border`, `width`) do **not** inherit by default, since inheriting them would rarely make sense (a child shouldn't automatically get its parent's exact margin).

**`!important` — Definition:** a modifier that overrides normal specificity/cascade rules, forcing that declaration to win regardless — generally considered a **last resort/anti-pattern**, since it breaks the predictable cascade and makes future overrides require *another* `!important`, escalating rather than resolving specificity conflicts.

**Units — Definition:**
- **`px`** — an absolute, fixed pixel value.
- **`%`** — relative to the parent element's corresponding dimension.
- **`em`** — relative to the *current element's* font size (compounds when nested — a common source of confusing sizing).
- **`rem`** — relative to the **root** (`<html>`) element's font size specifically, avoiding `em`'s nested-compounding problem — the generally preferred relative unit for consistent spacing/typography.
- **`vh`/`vw`** — relative to 1% of the viewport's height/width.
- **`ch`** — relative to the width of the `"0"` character in the current font, useful for constraining text line-length to a readable measure.

---

## 3. The Box Model

**Definition:** every rendered HTML element is treated as a rectangular box, composed of four nested layers, from innermost to outermost: **content** (the actual text/image), **padding** (space between content and border), **border** (a visible or invisible edge), and **margin** (space between this element's border and neighboring elements).

```
┌───────────────── margin ─────────────────┐
│  ┌───────────── border ─────────────┐    │
│  │  ┌─────────── padding ───────┐   │    │
│  │  │        content            │   │    │
│  │  └────────────────────────────┘   │    │
│  └────────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

**`box-sizing: content-box` vs `border-box` — Definition:** `content-box` (the CSS default) means a declared `width`/`height` applies **only to the content area** — padding and border are added *on top of* that, making the element's actual rendered size larger than the declared width. `border-box` means the declared `width`/`height` includes padding and border within it — the element's rendered size matches the declared value exactly, which is far more intuitive and is why nearly every modern CSS reset sets `box-sizing: border-box` globally.

```css
* { box-sizing: border-box; } /* common, near-universal reset */
```

**Margin collapsing — Definition:** when two block elements' **vertical** margins meet (e.g. one element's `margin-bottom` and the next element's `margin-top`), they don't add together — the larger of the two is used, not the sum — a frequently-surprising CSS behavior that only applies to vertical margins between block-level elements in normal flow (not horizontal margins, not margins on flex/grid items, not elements with padding/border between them).

**Display fundamentals — Definition:** the `display` property determines an element's fundamental rendering behavior. `block` (full-width by default, starts on a new line — `<div>`'s default), `inline` (flows with text, ignores `width`/`height` and top/bottom margin — `<span>`'s default), `inline-block` (flows with text like inline, but respects `width`/`height`/margin like block), `none` (removed from rendering **and** from layout entirely, as if it didn't exist).

**Visibility vs display — Definition:** `visibility: hidden` hides an element visually but **still reserves its layout space** (other elements don't shift to fill the gap); `display: none` removes the element from layout entirely (other elements shift to fill the space) — the choice depends on whether you want the gap to remain or collapse.

---

## 4. Layout Systems

**Normal document flow — Definition:** the default layout behavior — block elements stack vertically, inline elements flow horizontally within their line, without any explicit positioning applied — the baseline behavior that positioning/Flexbox/Grid all deliberately override.

**Positioning — Definition:**
- **`static`** (default) — normal flow; `top`/`left`/etc. have no effect.
- **`relative`** — stays in normal flow, but `top`/`left`/etc. offset it *visually* from where it would otherwise sit, without affecting surrounding elements' layout; also establishes a positioning context for any `absolute` descendants.
- **`absolute`** — removed from normal flow entirely, positioned relative to its nearest **positioned** ancestor (any ancestor with `position` other than `static`) — or the viewport if none exists.
- **`fixed`** — removed from normal flow, positioned relative to the **viewport**, staying in place even as the page scrolls.
- **`sticky`** — behaves like `relative` until the element would scroll out of a specified threshold (e.g. `top: 0`), at which point it "sticks" and behaves like `fixed` within its containing block.

**Stacking context & `z-index` — Definition:** `z-index` controls which element renders on top when two positioned elements overlap, but only compares elements within the **same stacking context** — certain CSS properties (`position` other than `static` + a `z-index` value, `opacity < 1`, `transform`, and others) create a *new* stacking context, meaning `z-index` values are only meaningfully compared against siblings within that same context, not globally across the whole page — a common source of "why isn't my `z-index: 9999` working" confusion.

**Flexbox — Definition:** a one-dimensional layout system for arranging items along a single axis (row or column), with powerful control over alignment, spacing, and how extra/insufficient space is distributed.

```css
.container {
  display: flex;
  justify-content: space-between; /* alignment along the MAIN axis */
  align-items: center;             /* alignment along the CROSS axis */
  flex-wrap: wrap;                 /* allow items to wrap to new lines */
  gap: 1rem;
}
.item {
  flex-grow: 1;    /* how much this item grows to fill extra space, relative to siblings */
  flex-shrink: 1;   /* how much this item shrinks when space is insufficient */
  flex-basis: 200px; /* the item's initial/preferred size before grow/shrink is applied */
}
```

**Main axis vs cross axis — Definition:** the **main axis** is the direction items are laid out along, set by `flex-direction` (`row` = horizontal main axis, the default; `column` = vertical); the **cross axis** is perpendicular to it — `justify-content` always aligns along the main axis, `align-items` always along the cross axis, regardless of `flex-direction`.

**CSS Grid — Definition:** a **two-dimensional** layout system for arranging items simultaneously across rows *and* columns — where Flexbox excels at distributing items along one line, Grid excels at defining an actual overall page/component grid structure.

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr); /* three equal-width columns */
  grid-template-rows: auto 1fr auto;      /* header/content/footer sizing */
  gap: 1rem;
}
.header { grid-column: 1 / -1; } /* span from the first to the last column line */
```

**Grid areas — Definition:** a more readable, visual way to define a grid layout by naming regions and arranging them like ASCII art:

```css
.container {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar content"
    "footer footer";
  grid-template-columns: 200px 1fr;
}
.header { grid-area: header; }
.sidebar { grid-area: sidebar; }
```

**Implicit vs explicit grid — Definition:** the **explicit** grid is what you define via `grid-template-columns`/`rows`; if content overflows that defined structure (more items than defined cells), the grid automatically creates additional **implicit** tracks to accommodate them, sized by `grid-auto-rows`/`grid-auto-columns` (default: `auto`-sized).

**Flexbox vs Grid — when to use which** — Flexbox for one-dimensional arrangements (a navbar's items, a row of buttons, distributing space along a single line); Grid for two-dimensional page/component layout (an overall page shell with header/sidebar/content/footer, a photo gallery grid) — in practice, most real UIs use **both together**: Grid for the outer page structure, Flexbox for arranging items within individual Grid areas.

---

## 5. Responsive Design

**Definition:** responsive design is the practice of building layouts that adapt to different viewport sizes/devices, using flexible layouts, relative units, and media queries — rather than building separate fixed layouts per device type.

**Mobile-first vs desktop-first — Definition:** **mobile-first** CSS writes base styles for the smallest/narrowest viewport, then uses `min-width` media queries to progressively add complexity/layout changes for larger screens; **desktop-first** does the reverse (`max-width` queries scaling down). Mobile-first is the broadly recommended default — it forces prioritizing essential content/functionality first, and tends to produce leaner CSS (progressively *adding* complexity rather than overriding it away).

**Media queries — Definition:** conditional CSS blocks applied only when specified conditions (typically viewport width) are met.

```css
/* mobile-first: base styles apply to ALL sizes, then override upward */
.container { display: block; }

@media (min-width: 768px) {
  .container { display: flex; }
}
```

**Fluid vs fixed layouts — Definition:** a **fixed** layout uses absolute pixel widths that don't adapt to the viewport; a **fluid** layout uses relative units (`%`, `fr`, `vw`) so the layout stretches/shrinks continuously with the viewport, rather than only changing at discrete media-query breakpoints.

**Container queries — Definition:** unlike media queries (which respond to the **viewport's** size), container queries let a component's styles respond to the size of its own **containing element** — enabling a genuinely reusable component that adapts its layout based on the space it's actually given, regardless of where it's placed on the page (a sidebar vs a full-width section) — a modern CSS feature (broadly supported since ~2023) solving a problem media queries fundamentally couldn't.

```css
.card-container { container-type: inline-size; }

@container (min-width: 400px) {
  .card { display: flex; } /* applies when the CONTAINER, not the viewport, is wide enough */
}
```

**Responsive images — Definition:** `srcset` lets the browser choose among multiple image resolutions based on the device's actual pixel density/viewport size, avoiding sending an oversized image to a small/low-DPI screen; `sizes` tells the browser how much space the image will actually occupy at different viewport widths, informing that choice; `<picture>` goes further, letting entirely different image *sources* (different crops, or modern formats like WebP with a fallback) be chosen based on media conditions.

```html
<img
  srcset="small.jpg 480w, medium.jpg 800w, large.jpg 1200w"
  sizes="(max-width: 600px) 100vw, 50vw"
  src="medium.jpg"
  alt="Description"
/>
```

**Breakpoint strategy** — rather than designing breakpoints around specific device widths (which change constantly across the device landscape), the more robust approach is to resize the browser and add a breakpoint wherever the *content itself* starts to look broken/cramped — content-driven breakpoints, not device-driven ones.

---

## 6. Typography & Visual Design in CSS

**Font properties — Definition:** `font-family` (an ordered fallback list of typeface names, ending in a generic family like `sans-serif`), `font-size`, `font-weight`, `font-style`, `line-height` — the core controls over how text renders.

**Web fonts (`@font-face`) — Definition:** loads a custom font file (not relying on fonts pre-installed on the user's device) for use via `font-family`.

```css
@font-face {
  font-family: 'MyFont';
  src: url('/fonts/myfont.woff2') format('woff2');
  font-display: swap; /* controls the FOUT/FOIT tradeoff, section 14 */
}
```

**Line height & vertical rhythm — Definition:** `line-height` controls the vertical space allotted to each line of text — a **unitless** value (e.g. `1.5`) is generally preferred over a fixed `px`/`em` value, since it scales proportionally with the element's own `font-size` rather than needing to be manually recalculated whenever font size changes.

**Color models — Definition:** `hex` (`#RRGGBB`, the traditional format); `rgb()`/`rgba()` (red/green/blue plus optional alpha transparency); `hsl()`/`hsla()` (hue/saturation/lightness — often more intuitive for programmatically deriving related colors, e.g. a lighter/darker variant by adjusting only lightness); `oklch()` (a newer, perceptually-uniform color space where equal numeric changes produce visually equal changes — increasingly used for generating consistent color scales/design tokens).

**CSS Custom Properties (Variables) — Definition:** author-defined, cascading, runtime-computed CSS values (`--variable-name`), referenced via `var()` — unlike Sass variables (section 9), which are resolved entirely at *compile time* and produce fully static output CSS, CSS Custom Properties are live in the browser: they can be changed dynamically via JavaScript or overridden per-element/media-query, and are the foundation of most modern theming/dark-mode implementations (section 18).

```css
:root { --primary-color: #3b82f6; --spacing-unit: 8px; }
.button { background: var(--primary-color); padding: calc(var(--spacing-unit) * 2); }

@media (prefers-color-scheme: dark) {
  :root { --primary-color: #60a5fa; } /* redefine the SAME variable — every usage updates automatically */
}
```

**Gradients, shadows, filters — Definition:** `linear-gradient()`/`radial-gradient()` generate color transitions without an image asset; `box-shadow` adds a drop shadow around an element's box; `filter` (`blur()`, `brightness()`, `grayscale()`) applies graphical effects, similar to an image-editing filter, to an entire element including its children.

**`clamp()` for fluid typography — Definition:** `clamp(min, preferred, max)` picks a value that scales fluidly with the preferred (often viewport-relative) value, while never going below the minimum or above the maximum — enables genuinely fluid, continuously-scaling font sizes across all viewport widths, without needing a discrete media-query breakpoint for every size step.

```css
h1 { font-size: clamp(1.5rem, 4vw + 1rem, 3rem); }
```

---

## 7. CSS Animations & Transitions

**Transitions — Definition:** smoothly animates a property's change **between two states** (e.g. its default state and a `:hover` state) over a specified duration, rather than the change happening instantaneously.

```css
.button {
  background: blue;
  transition: background-color 0.2s ease-in-out;
}
.button:hover { background: darkblue; }
```

**Keyframe animations (`@keyframes`) — Definition:** defines a sequence of style states at specified percentage points through an animation's duration, allowing more complex, multi-step, and self-triggering (not requiring a state change like `:hover`) animations than a transition can express.

```css
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
.spinner { animation: spin 1s linear infinite; }
```

**Transform — Definition:** `translate()`, `scale()`, `rotate()`, `skew()` visually reposition/resize/rotate/skew an element **without affecting document layout flow** (surrounding elements don't shift) — this is precisely why `transform` (and `opacity`) are the preferred properties to animate for performance (see below), unlike animating `width`/`top`/`margin`, which force the browser to recompute layout on every frame.

**Animation performance — Definition:** properties that can be handled entirely by the GPU **compositor** thread — `transform` and `opacity` — animate far more smoothly (and without blocking the main thread) than properties requiring layout recalculation (`width`, `height`, `top`, `left`, `margin`) or repaint (`background-color`, `box-shadow`) on every single frame — see section 14 for the full rendering-pipeline explanation of why.

**`will-change` — Definition:** a hint telling the browser to prepare an optimized rendering path (often promoting the element to its own GPU layer) *before* an animation starts, avoiding a jank/stutter on the animation's first frame — should be used sparingly and removed once the animation is done, since keeping too many elements on separate GPU layers indefinitely has its own memory/performance cost.

**Reduced motion — Definition:** `@media (prefers-reduced-motion: reduce)` detects a user's OS-level accessibility preference to minimize animation (often set by users prone to motion sickness/vestibular disorders), letting a site disable or simplify non-essential animations for those users — an accessibility consideration (section 13) specific to motion design.

```css
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```

---

## 8. CSS Architecture & Methodologies

**Why CSS architecture matters at scale — Definition:** CSS has no built-in namespacing/scoping (every selector is globally applicable to the whole page by default), so as a codebase grows, unrelated styles increasingly clash/override each other unpredictably — CSS architecture methodologies exist specifically to impose a disciplined naming/organization convention that avoids this, in the absence of language-level scoping.

**BEM (Block Element Modifier) — Definition:** a naming convention: `.block__element--modifier` — a **Block** is a standalone, reusable component (`.card`); an **Element** is a part of that block, tied to it (`.card__title`); a **Modifier** is a variant/state of a block or element (`.card--featured`).

```html
<div class="card card--featured">
  <h2 class="card__title">Title</h2>
  <p class="card__body">Body text</p>
</div>
```
BEM's flat, explicit naming avoids nested selectors (which raise specificity and create fragile parent-child coupling) and makes a class's role and relationships obvious purely from its name.

**OOCSS (Object-Oriented CSS) — Definition:** a methodology emphasizing separating **structure from skin** (layout/sizing rules separate from visual/decorative rules) and reusing small, composable classes across multiple different components, rather than one large, component-specific class per component.

**SMACSS (Scalable and Modular Architecture for CSS) — Definition:** categorizes styles into five types — **Base** (element defaults, no classes), **Layout** (major page regions), **Module** (reusable components), **State** (a module's state, like `.is-active`), **Theme** (visual overrides) — providing an organizational framework for *where* different kinds of styles should live in a codebase.

**Utility-first CSS (philosophy) — Definition:** instead of writing custom, semantically-named CSS classes per component (BEM's approach), compose UI directly from small, single-purpose utility classes applied straight in the markup (`class="flex items-center gap-2 p-4"`) — Tailwind CSS (section 10) is the dominant modern implementation of this philosophy.

**CSS Modules — Definition:** a build-time tool (integrated into most modern bundlers) that automatically scopes CSS class names to the specific component file that imports them (by generating unique, hashed class names under the hood), eliminating global-namespace collisions **without** needing a manual naming convention like BEM to achieve it.

```css
/* Card.module.css */
.title { font-weight: bold; } /* compiles to something like ._title_a8f3c */
```
```jsx
import styles from './Card.module.css';
<h2 className={styles.title}>Title</h2>
```

**CSS-in-JS (styled-components, emotion) — Definition:** writing component styles directly in JavaScript/TypeScript, colocated with the component itself, with the library generating scoped CSS (often with unique class names, similar to CSS Modules) at build or runtime, and supporting dynamic styling directly from component props/state.

```jsx
const Button = styled.button`
  background: ${props => (props.primary ? 'blue' : 'gray')};
  padding: 0.5rem 1rem;
`;
```
CSS-in-JS's key advantage is tight colocation with component logic and props-driven dynamic styling; its tradeoffs include added runtime/bundle cost for libraries that generate styles at runtime (rather than compiling them away at build time, as newer "zero-runtime" CSS-in-JS tools aim to do) and a learning curve for teams used to plain CSS/Sass.

**Choosing an architecture** — BEM/SMACSS for plain-CSS projects without a component-based build system; CSS Modules or CSS-in-JS for component-framework (React/Angular/Vue) projects wanting automatic scoping; utility-first (Tailwind) for teams that value rapid iteration directly in markup over maintaining a separate stylesheet — the right choice depends heavily on the project's tooling and team preference more than any objectively "correct" answer.

---

## 9. Sass / SCSS

A CSS preprocessor deep-dive.

**Definition:** Sass (Syntactically Awesome Style Sheets) is a CSS preprocessor — a language that compiles down to plain CSS, adding programming-language features (variables, nesting, functions, control flow) that plain CSS historically lacked, authored in either the original indentation-based `.sass` syntax or the more common, CSS-compatible `.scss` syntax.

**Why preprocessors exist** — before CSS Custom Properties (section 6) and native nesting (section 16) existed in browsers, Sass was the primary way to get variables, nesting, and reusable logic in stylesheets at all — much of Sass's original value proposition has since been absorbed into native CSS, though Sass retains capabilities (real compile-time functions/control-flow/mixins) native CSS still doesn't have.

**Variables:**

```scss
$primary-color: #3b82f6;
$spacing: 8px;

.button { background: $primary-color; padding: $spacing * 2; }
```

**Nesting — Definition:** write child selectors nested inside their parent's rule block, mirroring the HTML structure, compiling down to flattened, fully-qualified CSS selectors.

```scss
.card {
  padding: 1rem;
  &__title { font-weight: bold; }     // & refers to the parent selector — compiles to .card__title
  &:hover { box-shadow: 0 0 4px #000; }
  .dark-mode & { background: #222; }   // nesting can also reference an ANCESTOR context
}
```

**Partials & `@use`/`@forward` — Definition:** a **partial** (`_variables.scss`, the leading underscore excludes it from being compiled to its own separate CSS file) holds shared variables/mixins imported into other files; `@use` (the modern module system) imports a partial with proper namespacing (avoiding global naming collisions, unlike the legacy `@import`, which dumps everything into one global namespace and is now deprecated); `@forward` re-exports another partial's members from a single entry-point file.

**Mixins — Definition:** a reusable block of CSS declarations, optionally parameterized, inserted wherever `@include`d — for CSS patterns that repeat across many rules with slight variation.

```scss
@mixin flex-center($direction: row) {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: $direction;
}
.container { @include flex-center(column); }
```

**Functions — Definition:** unlike mixins (which output CSS declarations), a Sass `@function` returns a **value**, usable anywhere a value is expected — e.g. a function computing a color's lighter/darker variant.

**`@extend` — Definition:** lets one selector inherit all the declarations of another, causing the compiler to combine both selectors under one shared rule in the output CSS — differs from a mixin (which duplicates the declarations into every including selector) by instead grouping selectors together; generally used more cautiously than mixins today, since `@extend` can produce unexpectedly large, hard-to-predict combined selectors, especially across partials.

**Control directives — Definition:** `@if`/`@else`, `@each` (loop over a list/map), `@for` (loop a fixed number of times) — genuine compile-time programming constructs, used for generating repetitive CSS patterns (e.g. a full utility-class scale) programmatically rather than writing each variant by hand.

```scss
$spacers: (1: 4px, 2: 8px, 3: 16px);
@each $key, $value in $spacers {
  .p-#{$key} { padding: $value; }
}
```

**Sass vs plain CSS Custom Properties today** — CSS Custom Properties (section 6) now natively cover simple variable use cases, and native CSS nesting (section 16) now covers basic nesting — but Sass retains real, still-unmatched-natively advantages: genuine compile-time functions/loops/conditionals, and `@use`/`@forward`'s proper module namespacing — many modern projects use *both* together (Sass for its programming-language features, CSS Custom Properties specifically for values that need to be dynamic *at runtime*, like theme switching).

---

## 10. Tailwind CSS

A utility-first framework deep-dive.

**Definition:** Tailwind CSS is a utility-first CSS framework providing a large, constrained set of low-level utility classes (`flex`, `pt-4`, `text-center`, `bg-blue-500`) that are composed directly in markup to build custom designs, rather than providing pre-built, opinionated components (contrast with Bootstrap, section 11).

**Core concepts:**

```html
<button class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 md:px-6">
  Click me
</button>
```
- **Utility classes** — each class does exactly one thing (`px-4` = horizontal padding, `bg-blue-500` = a specific background color from Tailwind's default color scale) — styling is composed by combining many small utilities rather than writing custom CSS rules.
- **Responsive variants** — a `breakpoint:` prefix (`md:px-6`) applies a utility only from that breakpoint upward (Tailwind is mobile-first by default, section 5) — no separate media-query block needed in a stylesheet.
- **State variants** — a `state:` prefix (`hover:bg-blue-600`, `focus:ring-2`, `disabled:opacity-50`) applies a utility only in that pseudo-class/state, again inline, without writing a separate `:hover` rule.

**Configuration (`tailwind.config.js`) — Definition:** the file where a project's **design tokens** (section 11) — color palette, spacing scale, font families, breakpoints — are customized/extended, so utility classes reflect the project's actual design system rather than Tailwind's generic defaults.

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: { brand: '#3b82f6' },
      spacing: { 18: '4.5rem' },
    },
  },
};
```

**`@apply` (and when to avoid it) — Definition:** lets you pull a group of utility classes into a single custom CSS class (`@apply flex items-center gap-2;` inside a `.btn` rule) — useful for a repeated pattern that appears identically in many places, but overusing `@apply` throughout a codebase gradually reintroduces the exact custom-CSS-class maintenance burden utility-first CSS was meant to avoid in the first place; generally reserved for genuinely repeated, stable low-level patterns rather than every component.

**Component extraction patterns** — in a component-based framework (React/Angular/Vue), the idiomatic way to avoid repeating long utility class strings isn't `@apply` — it's extracting a reusable **component** (`<Button variant="primary">`) whose implementation contains the utility classes once, reused via the component API rather than copy-pasted class strings.

**JIT compilation & purging unused CSS — Definition:** Tailwind's Just-In-Time engine scans your actual source files for utility classes that are genuinely used, and generates **only those** classes' CSS at build time — rather than shipping the framework's entire (enormous) set of possible utility combinations, keeping production CSS bundle size small regardless of how large Tailwind's total utility vocabulary is.

**Tailwind vs traditional CSS — tradeoffs:**
- **Advantage:** extremely fast iteration (no context-switching between markup and a separate stylesheet, no need to invent class names for every new element), naturally constrains design decisions to the configured design-token scale (preventing "one-off" inconsistent values from creeping in), and small final CSS bundle via JIT purging.
- **Disadvantage:** markup becomes visually dense with utility classes, which some find harder to read at a glance than a semantic class name; genuinely custom/one-off visual designs that don't fit the utility vocabulary well can require dropping into arbitrary-value syntax (`w-[137px]`) or custom CSS anyway; a real learning curve memorizing/looking up the utility class vocabulary.

---

## 11. Component & Design-System Frameworks

**Bootstrap — Definition:** a comprehensive CSS framework providing a responsive **grid system**, and a large library of **pre-styled, ready-to-use components** (navbars, modals, cards, buttons) with a consistent, immediately-recognizable default visual style — the original, still widely-used "batteries-included" CSS framework.

```html
<div class="container">
  <div class="row">
    <div class="col-md-6">Half width on medium+ screens</div>
    <div class="col-md-6">Half width on medium+ screens</div>
  </div>
</div>
```
**Advantage:** extremely fast to get a fully-functional, consistent-looking UI without designing from scratch. **Disadvantage:** sites built with default Bootstrap styling are visually recognizable/generic unless meaningfully customized, and its pre-built components' JS behavior can be harder to integrate cleanly with a component framework's own state management (React/Angular) than a headless approach (below).

**Material UI / Material Design — Definition:** **Material Design** is Google's design language/specification (elevation via shadows, specific motion/interaction principles, a defined component behavior spec); **Material UI (MUI)** is a popular React component library implementing that specification as ready-to-use, fully-styled React components.

**Headless UI libraries (Radix, Headless UI) — Definition:** component libraries that provide fully accessible, correctly-behaving **logic and interaction patterns** (keyboard navigation, focus management, ARIA attributes, open/close state) for complex components (dropdowns, modals, comboboxes) **without any bundled visual styling** — you supply your own CSS/Tailwind classes, getting the hard, easy-to-get-wrong accessibility/interaction logic for free while retaining full visual control (unlike Bootstrap/MUI's opinionated look).

**Design tokens — Definition:** the smallest, named, reusable values in a design system — colors, spacing units, font sizes, border radii — defined once (often as CSS Custom Properties or a Tailwind config, sections 6/10) and referenced everywhere, rather than hardcoded per-component, so a design change propagates from one source of truth.

**Building vs adopting a design system** — adopting an existing one (Bootstrap, MUI) is faster to start and battle-tested, but couples the product's visual identity to that library's look/constraints; building a custom design system (often on top of a headless library + Tailwind/CSS Modules for styling) gives full design control and a distinctive product identity, at significantly more upfront and ongoing engineering investment — the right choice scales with how much visual differentiation/brand identity actually matters for the product.

---

## 12. CSS Layout Patterns & Common UI Components

**Centering techniques:**

```css
/* flexbox — the most common modern approach, centers in BOTH directions */
.parent { display: flex; align-items: center; justify-content: center; }

/* grid — equally simple, one-liner centering */
.parent { display: grid; place-items: center; }

/* absolute positioning + transform — for centering within a specific positioned ancestor */
.child { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); }

/* text-only horizontal centering */
.text { text-align: center; }
```

**Sticky headers/footers — Definition:** `position: sticky` (section 4) on a header keeps it pinned to the top of the viewport once scrolled to, without removing it from document flow the way `fixed` does — the modern, simplest approach, superseding older JS-scroll-listener-based sticky-header implementations.

**Holy grail layout — Definition:** the classic three-column layout (header, footer, and a main content area flanked by a left and right sidebar) — historically notoriously fiddly with floats, now trivially expressed with CSS Grid's named grid areas (section 4).

**Card layouts — Definition:** a common UI pattern — a contained, elevated (often via `box-shadow`) rectangular block grouping related content — typically laid out in a responsive grid/flex-wrap arrangement (`display: grid; grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));` is a common one-liner for a responsive card grid that automatically fits as many cards per row as space allows, with no media queries needed).

**Responsive navigation patterns** — common approach: a horizontal nav bar on wide viewports collapses into a "hamburger" icon triggering an off-canvas or dropdown menu below a defined breakpoint — implemented with a media query controlling visibility/layout, plus (for the toggle interaction itself) a small amount of JS to add/remove an "open" class.

**Modal/dialog styling considerations** — beyond visual styling: proper modals need a backdrop overlay, must trap focus within themselves while open (accessibility, section 13), must be dismissible via `Escape`, and should use the native `<dialog>` element or a headless library (section 11) to get this behavior correctly rather than reimplementing it from scratch.

**CSS-only vs JS-assisted components — Definition:** some interactive-seeming components can be built with **pure CSS** — an accordion/tooltip using the `:checked` pseudo-class trick with a hidden checkbox, or the native `<details>`/`<summary>` elements — avoiding any JS dependency for simple cases, though JS-assisted implementations (or a headless library) are generally preferable once genuine accessibility/keyboard-interaction requirements exceed what the CSS-only trick can express cleanly.

---

## 13. Accessibility (a11y)

**Definition:** web accessibility is the practice of designing and building websites usable by people with disabilities — visual, auditory, motor, or cognitive — ensuring the site works with assistive technology (screen readers, keyboard-only navigation, voice control) and doesn't rely on a single sense/interaction mode alone.

**Semantic HTML as the foundation (recap)** — see section 1; using the correct native element (`<button>` instead of a `<div onclick>`) gets an enormous amount of accessibility behavior (keyboard focusability, correct screen-reader announcement, native `Enter`/`Space` activation) entirely for free, without any extra code.

**ARIA roles, states, and properties — Definition:** ARIA (Accessible Rich Internet Applications) attributes supplement HTML's native semantics **only when a native element can't express what's needed** — `role` (declares an element's purpose to assistive tech, e.g. `role="alert"`), states (`aria-expanded`, `aria-checked` — dynamic, reflect current state), properties (`aria-label`, `aria-describedby` — mostly static). The first rule of ARIA is: **use a native semantic element instead, if one exists** — ARIA is a supplement for genuinely custom widgets (a custom dropdown, a custom tab panel), not a first resort.

```html
<button aria-expanded="false" aria-controls="menu">Menu</button>
<ul id="menu" role="menu" hidden>...</ul>
```

**Keyboard navigation & focus management — Definition:** every interactive element must be reachable and operable via keyboard alone (`Tab` to move focus, `Enter`/`Space` to activate) — native interactive elements (`<button>`, `<a href>`, form controls) get this automatically; custom interactive elements (a `<div>` acting as a button) need explicit `tabindex="0"` and manually-wired keyboard event handlers to match — another reason to prefer native elements wherever possible.

**Focus indicators & `:focus-visible` — Definition:** a visible outline/highlight showing which element currently has keyboard focus — essential for keyboard users to track their position on the page. `:focus-visible` (as opposed to plain `:focus`) lets you show the focus ring specifically for keyboard navigation while suppressing it for mouse clicks (where a visible focus ring is often considered visually unnecessary) — solving the older, accessibility-harmful practice of `outline: none` removing focus indicators entirely for everyone.

**Color contrast — Definition:** WCAG (Web Content Accessibility Guidelines) defines minimum contrast ratios between text and its background (e.g. 4.5:1 for normal text at the AA level) to ensure readability for users with low vision or color blindness — checked with browser DevTools' contrast checker or dedicated tools, not by eye.

**Screen reader considerations** — meaningful `alt` text on images (empty `alt=""` specifically for purely decorative images, so screen readers skip them rather than announcing a meaningless filename); a logical heading hierarchy (`h1` → `h2` → `h3`, not skipped levels) that screen reader users navigate by; visually-hidden-but-screen-reader-accessible text (a `.sr-only` utility class) for context that's visually implied but not literally present in the text.

**Accessible forms** — every input needs an associated, visible `<label>` (via `for`/`id`, or wrapping the input) — a placeholder is **not** an adequate substitute for a label (it disappears once typing starts, and isn't reliably announced by all screen readers); validation errors should be programmatically associated with their field (`aria-describedby`) and announced (`aria-live` or `role="alert"`), not conveyed by color alone.

**Testing accessibility — Definition:** automated tools (`axe`, Lighthouse's accessibility audit) catch a meaningful subset of issues (missing alt text, insufficient contrast, missing form labels) but **cannot** catch everything (logical reading order, whether ARIA usage is actually semantically correct, genuine usability with a real screen reader) — automated tools are a useful baseline floor, not a substitute for actual manual keyboard-only and screen-reader testing.

---

## 14. Browser Rendering & CSS Performance

**The browser rendering pipeline — Definition:** the sequence of steps a browser performs to turn HTML/CSS into pixels on screen:

```
Parse HTML → build DOM
Parse CSS → build CSSOM
DOM + CSSOM → Render Tree (visible elements + their computed styles)
      ↓
Layout ("Reflow") — compute exact size/position of every element
      ↓
Paint — fill in pixels (color, text, images, borders) for each element
      ↓
Composite — combine painted layers together, applying transforms/opacity, onto the screen
```

**Reflow vs repaint vs composite — Definition:**
- **Reflow (layout)** — triggered by any change affecting an element's geometry (size, position) — the most expensive, since it can cascade: changing one element's size can force recalculating the position of many other elements on the page.
- **Repaint** — triggered by a visual-only change that doesn't affect layout (e.g. `background-color`, `visibility`) — skips the layout step, but still redraws pixels.
- **Composite** — triggered by changes to properties the compositor thread can handle entirely on its own (`transform`, `opacity`) — the cheapest, and the reason these are the recommended properties to animate (section 7).

**Expensive CSS properties** — properties that force layout (`width`, `height`, `top`, `left`, `margin`, `padding`, adding/removing DOM elements) are the most expensive to animate/change frequently; properties that only force repaint (`color`, `background-color`, `box-shadow`) are cheaper but still avoidable; `transform`/`opacity` changes are cheapest, handled by composite alone.

**Critical CSS — Definition:** the minimal subset of CSS actually needed to render the content visible **above the fold** (without scrolling) on initial page load — inlined directly into the HTML's `<head>` so the browser can render that visible content immediately, without waiting for a full external stylesheet to download, while the remainder of the CSS loads asynchronously in the background.

**CSS specificity performance implications** — modern browser CSS selector matching is fast enough that specificity/selector complexity is rarely a *meaningful* real-world performance bottleneck on its own; the practical cost of an overly-complex specificity architecture is almost always **maintainability** (fighting the cascade, needing `!important`) rather than raw runtime CSS-matching performance.

**Reducing unused CSS** — shipping CSS the current page never actually uses adds unnecessary download/parse weight; Tailwind's JIT purging (section 10) is one automated solution; more generally, browser DevTools' Coverage tab reports exactly which loaded CSS rules were actually applied on a given page, surfacing genuinely dead/unused styles.

**Font loading performance (FOUT/FOIT/FOFT) — Definition:** while a custom web font (section 6) is loading, the browser must decide what to do with text using it:
- **FOIT** (Flash of Invisible Text) — text is hidden entirely until the font loads (the old browser default) — can leave content invisible if the font is slow/fails to load.
- **FOUT** (Flash of Unstyled Text) — text renders immediately in a fallback font, then swaps to the custom font once loaded, causing a visible re-layout/flash (`font-display: swap` explicitly opts into this behavior).
- **FOFT** (Flash of Faux Text) — a more advanced strategy loading a font in stages (a lightweight version first, the full version later), minimizing both invisible text and jarring reflow.

Generally, `font-display: swap` (accepting a brief FOUT) is preferred over the invisible-text default, since showing *readable* fallback text immediately is almost always a better user experience than showing nothing.

---

## 15. Cross-Browser & Compatibility

**Browser support strategy — Definition:** deliberately defining which browsers/versions a project needs to support (often informed by real analytics of the actual user base) — a decision that then determines which CSS features are safe to use directly vs need a fallback/polyfill, rather than either recklessly using every new feature or overly conservatively avoiding anything not universally supported for years.

**Vendor prefixes & Autoprefixer — Definition:** browser engines historically shipped experimental CSS features behind prefixes (`-webkit-`, `-moz-`, `-ms-`) before standardizing unprefixed syntax; **Autoprefixer** (a PostCSS plugin, section 17) automatically adds whatever prefixes are still needed based on a configured browser support target, so developers write plain, unprefixed modern CSS and let tooling handle the compatibility layer — writing vendor prefixes by hand today is essentially obsolete practice.

**Feature detection (`@supports`) — Definition:** a native CSS at-rule that conditionally applies a block of styles only if the browser actually supports a given property/value — enables genuinely progressive enhancement (below) directly in CSS, without needing JavaScript-based feature detection for styling concerns.

```css
.container { display: block; } /* fallback for browsers without grid support */

@supports (display: grid) {
  .container { display: grid; }
}
```

**Progressive enhancement vs graceful degradation — Definition:** **progressive enhancement** starts from a baseline, universally-working experience and layers on enhancements for browsers that support them (the `@supports` example above); **graceful degradation** starts from the full, modern experience and adds fallbacks for older browsers as an afterthought — progressive enhancement is generally considered the more robust philosophy, since it guarantees a working baseline by construction rather than by retrofitting.

**Polyfills (brief) — Definition:** JavaScript code that implements a missing browser feature/API so it behaves as if natively supported — primarily relevant for JS APIs; genuinely missing *CSS* features generally cannot be polyfilled in the same way (a CSS feature the rendering engine doesn't understand can't be made to work via JS alone), which is why `@supports`-based fallback styling is CSS's own equivalent strategy.

**Testing across browsers/devices** — real-device testing (or cloud device-testing services like BrowserStack) catches genuine rendering/interaction differences that browser DevTools' device-emulation mode can approximate but not perfectly replicate (real touch behavior, real OS-level font rendering, real performance characteristics).

---

## 16. Modern CSS Features

**CSS Nesting (native) — Definition:** browsers now natively support nested selectors (broadly supported since 2023), similar in spirit to Sass's nesting (section 9) but running directly in the browser with no build step/compiler required.

```css
.card {
  padding: 1rem;
  & .title { font-weight: bold; }
  &:hover { box-shadow: 0 0 4px #000; }
}
```

**`:has()` (the parent selector) — Definition:** a long-requested CSS capability — selects an element **based on its descendants/relatives matching a condition** — effectively a "parent selector," something CSS previously had no way to express at all.

```css
/* style a .card that CONTAINS an image, differently from one that doesn't */
.card:has(img) { display: grid; grid-template-columns: 100px 1fr; }
```

**Container queries (recap)** — see section 5; component-level, not viewport-level, responsive styling.

**Cascade layers (`@layer`) — Definition:** lets you explicitly define named layers of CSS, with layer **order** (not specificity or source order) determining precedence between layers — styles in a later-declared layer always beat an earlier layer's styles, *regardless* of selector specificity within them — a powerful tool for cleanly layering, e.g., a third-party component library's base styles beneath your own application overrides, without a specificity arms race.

```css
@layer base, components, utilities;

@layer base { h1 { font-size: 2rem; } }
@layer utilities { .text-center { text-align: center; } } /* always wins over `base`, regardless of specificity */
```

**Logical properties — Definition:** properties expressed relative to writing-mode/text-direction (`margin-inline-start` instead of `margin-left`, `padding-block` instead of `padding-top`/`padding-bottom` combined) — automatically adapt correctly for right-to-left languages (Arabic, Hebrew) or vertical writing modes, without needing separate direction-specific CSS overrides.

**Subgrid — Definition:** lets a nested grid item inherit its **parent's** grid track sizing for its own internal grid, rather than defining independent tracks — solves the common problem of wanting nested grid items (e.g. cards in a grid, each with their own internal grid) to align their internal content across siblings, which plain nested Grid couldn't achieve on its own.

**`color-mix()` — Definition:** a native CSS function for blending two colors together at a specified ratio, directly in CSS — enables generating derived colors (a lighter/darker/tinted variant) without needing Sass color functions (section 9) or a JS-based design-token build step for simple cases.

```css
.button:hover { background: color-mix(in srgb, var(--primary-color) 80%, black); }
```

**Scroll-driven animations (brief) — Definition:** a newer CSS capability tying an animation's progress directly to scroll position (a page's scroll, or a specific scrollable element's scroll) rather than to elapsed time — enables effects like a progress bar that fills as the user scrolls, or a parallax effect, entirely in CSS without a JS scroll-event listener.

---

## 17. Build Tooling for CSS

**PostCSS & plugins — Definition:** a tool that transforms CSS via a plugin pipeline — Autoprefixer (section 15) is one common plugin; others include `postcss-preset-env` (lets you write tomorrow's CSS syntax today, compiling it down for current browser support), minifiers, and custom transform plugins — PostCSS itself is just the plugin-execution engine, not a fixed feature set.

**CSS bundling (Vite/webpack handling of CSS) — Definition:** modern bundlers process `import './styles.css'` statements directly in JS/component files, resolving, combining, and (in production) minifying/hashing the resulting CSS output automatically — the same bundling concepts already covered in the React notes' production-engineering section, applied specifically to CSS assets.

**Minification & optimization** — stripping whitespace/comments and applying various CSS-specific optimizations (merging duplicate rules, shortening color values) to reduce the shipped CSS file's size — handled automatically by the production build step of virtually every modern bundler/tool.

**Source maps for CSS — Definition:** mapping between the compiled/minified CSS actually served to the browser and its original source (a Sass file, a CSS Module) — lets browser DevTools show and let you edit styles against the *original* source location, rather than an unreadable, minified, or Sass-compiled output file.

**Linting (Stylelint) — Definition:** a linter for CSS/Sass, catching errors (invalid property values, duplicate selectors) and enforcing style consistency (property ordering, naming conventions) — the CSS-world equivalent of ESLint for JavaScript, integrable into the same CI pipeline described in the Node.js/Docker notes.

**Design token pipelines — Definition:** tooling (e.g. Style Dictionary) that takes design tokens defined once in a platform-agnostic format (often JSON) and generates platform-specific output — CSS Custom Properties for web, Sass variables, or even native mobile platform formats — from that single source of truth, keeping design values consistent across every platform a design system needs to reach.

---

## 18. Production Engineering

**CSS bundle size & code splitting — Definition:** as with JS (React notes' section 12), CSS can be split so a given page/route only loads the CSS it actually needs, rather than one enormous global stylesheet shipped on every page — most component-based bundlers handle this automatically when CSS is imported per-component/per-route (CSS Modules, CSS-in-JS) rather than as one global file.

**Critical CSS extraction strategies (recap)** — see section 14; automated tooling exists (e.g. `critical`, or built into some SSR frameworks) to extract and inline above-the-fold CSS automatically at build/render time, rather than manually curating it.

**Caching strategy for CSS assets** — the same hashed-filename + far-future-cache-header strategy already covered in the Angular/React/Node.js production notes applies identically to CSS bundle files — `styles.a1b2c3.css` can be cached indefinitely, since any content change produces a new hash/filename.

**Dark mode implementation strategies — Definition:** the modern standard approach layers CSS Custom Properties (section 6) with `@media (prefers-color-scheme: dark)` for automatic OS-preference detection, plus (for a manual in-app toggle, overriding the OS preference) a `data-theme` attribute on the root element that a small amount of JS toggles, with CSS selectors like `:root[data-theme="dark"]` taking precedence — the same three-way light/dark/system pattern already covered in this workspace's Artifact-authoring guidance, applied here to a general production website.

```css
:root { --bg: white; --text: black; }

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) { --bg: #111; --text: white; }
}
:root[data-theme="dark"] { --bg: #111; --text: white; }
```

**Theming at scale** — for a design system serving multiple themes/brands (not just light/dark), the same Custom Properties approach scales by swapping an entire set of token values based on a top-level theme class/attribute, keeping every component's own CSS theme-agnostic (referencing only `var(--token-name)`, never a hardcoded value).

**Common pitfalls & anti-patterns:**
- Overly-specific/deeply-nested selectors that fight the cascade and require `!important` to override later.
- Using tables or absolute positioning for general page layout instead of Flexbox/Grid.
- Missing `box-sizing: border-box`, causing padding/border to unexpectedly enlarge declared widths.
- Building custom interactive components (dropdowns, modals) from scratch without proper keyboard/ARIA support instead of using a headless library.
- Animating layout-triggering properties (`width`, `top`, `margin`) instead of `transform`/`opacity`, causing janky animations.
- Shipping an entire unused CSS framework instead of only what a page actually needs (unpurged Bootstrap/Tailwind).
