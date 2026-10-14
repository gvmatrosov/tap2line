# Stacks — when tap2line fits, and where the locator lives

## When a compiler already knows the line

If the UI is a component framework compiled by a dev build (React, Vue, Svelte, Solid, Angular
with Vite/webpack), the compiler can stamp the exact source position onto every element. Tools
built on that give exact answers and usually open the editor at the line — for example
LocatorJS or click-to-react-component for React, the Vue DevTools / vite-plugin-vue-inspector
for Vue, the inspector built into `@sveltejs/vite-plugin-svelte` for Svelte, and
`code-inspector-plugin`, which covers several frameworks. Check what the project already has
before recommending one.

Tell the person that in one or two lines. tap2line can still make sense there — its answer is a
clipboard line for Claude, it works from a phone with no editor, it needs no build plugin — but
then its corpus is the component source (`.jsx`, `.vue`, `.svelte`), and the birth pattern must
match `className="…"`, `:class="…"`, `class:x=` and the like.

## Where tap2line is at home

| markup is born in | corpus | birth pattern adds |
|---|---|---|
| JS template strings (`innerHTML`, `insertAdjacentHTML`, lit-html) | the client JS | `class="…"` inside backticks |
| DOM built by hand | the client JS | `className =`, `classList.add(`, `setAttribute('class'` |
| plain `.html` files | the HTML | `class="…"`, `id="…"` |
| server templates (EJS, Jinja, Blade, ERB, Go templates, Twig, PHP) | the template directory (+ client JS) | the template's own class syntax, e.g. `class="{{ … }} x"` |
| htmx / Alpine on server templates | templates + client JS | as above; `hx-*`/`x-*` attributes are useful `data`-like tokens |

## Where the locator runs

- **The project has a server in JS/TS** → a dev-only route module in the project's routing style,
  mounted next to the others.
- **The server is another language** (Python, Ruby, PHP, Go…) → write the locator in that
  language as a dev-only route; the algorithm is plain string scanning. If that is heavy, use the
  standalone script below and say why.
- **No server** (static files, a plain `python -m http.server`) → a standalone zero-dependency
  Node script (`node tools/tap2line.js`) on its own localhost port, serving only `ping` and
  `locate`. It allows CORS only from the app's own dev origin, never `*`. The client then calls
  that origin; if the app is served over https, the script needs https too or the browser blocks
  the call as mixed content.

The client is the same in all three cases; only the base URL of `ping`/`locate` changes.
