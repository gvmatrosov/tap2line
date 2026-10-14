---
name: setup
description: Build a tap-to-source-line inspector into this web project — a toggle button in the running app; while it is on, tapping any element puts the file and line that builds it on the clipboard (`public/cart.js:212 · handler :240`), ready to paste into Claude Code. Use when the person asks for a code inspector, "click an element → find its code", "where is this button in the source", tap2line, or keeps describing on-screen things in words so every session searches for them again. Made for markup born in hand-written JS template strings, plain HTML or server templates — no build step needed.
---

# tap2line — tap an element, get its source line

You are going to build, inside this person's project, a small dev tool with two halves:

- **client** — a button in the running app. While the mode is on, a tap does not press anything;
  it reads the tapped element's signature (tag, id, classes, `data-*`, own text, plus up to five
  ancestors and the nearest container id) and sends it to the locator;
- **locator** — a dev-only endpoint that scores every line of the client source against that
  signature and answers with one address, or honestly with `(?)` and the alternatives.

The answer lands on the clipboard as one line: `public/cart.js:212 · handler :240`. The person
pastes it into the prompt; you start from the line instead of searching.

**The address is computed at request time from the live source. Nothing is stored or stamped
into the markup**, so nothing can rot when a line is inserted above. This is the whole idea; do
not trade it away for a lookup table or `data-src="file:line"` attributes written by hand.

This skill is a request, not a lock: nothing checks that you followed it. The guards you build in
step 5 are what make the result true — the tool is **not done** until they are green and have been
seen to go red.

## 0. Is this the right tool here?

Read [references/stacks.md](references/stacks.md). In short: if the UI is React/Vue/Svelte/Solid
compiled by a dev build, the compiler already knows the exact source line of every element, and
a build-time tool gives exact answers where this one gives ranked guesses. Say so in one or two
lines, name the tool from stacks.md, and build tap2line there only if the person still wants it
(for example: a phone with no editor, or the clipboard line for Claude is the point).

tap2line's home is where no compiler knows the line: markup built in template strings
(`` el.innerHTML = `<div class="card">…` ``), DOM built by hand (`el.className = 'card'`), plain
`.html`, server templates (EJS, Jinja, Blade, ERB, Go templates, PHP).

## 1. Look at the project — detect first, ask only what cannot be seen

Find out, by reading, not asking:

1. **Where the markup is born** — which directories hold the files that build what the browser
   shows. That set is the **corpus**. Usually: the static/public JS and HTML, plus the template
   directory if the server renders HTML. Not: `node_modules`, vendored and `*.min.js` files,
   build output (`dist/`, `build/`), media/data folders.
2. **How elements are born** — `class="x"` in strings, `className =`, `classList.add(`,
   `setAttribute('class'`, a templating helper of the project's own. The locator's "birth line"
   pattern must match what this codebase actually writes (references/locator.md, § birth).
3. **The app root and the screen containers** — the element the app renders into (`#app`,
   `#view`, `main`) and the ids of the containers each screen draws into. The container id is
   the locator's strongest file filter.
4. **The server** — is there one, in which language, how are routes mounted, how does it tell
   development from production (env flag, `NODE_ENV`, a config), and is there a login whose
   "owner/admin" check you can reuse.
5. **Clicks already taken** — right-click menus, drag that starts on `pointerdown`, a global
   keyboard shortcut that would clash with Alt+I.
6. **Tests** — how the project runs tests, and whether a headless browser (Playwright,
   Puppeteer) is already installed.

## 2. Show the plan and wait for an ok

Before writing code, give the person a short plan in plain words:

- the corpus (directories) and what is left out of it, and why;
- where the locator lives: a dev-only route in their server, or — with no server, or a server in
  another language you would rather not touch — a standalone zero-dependency Node script
  (stacks.md);
- **the gate**: who can reach it (references/locator.md, § gate). If the dev server is reachable
  from the internet, say so plainly — a flag on a public server is not a dev/prod boundary;
- the files you will add (client JS, its CSS, the locator, the cases, the guard runners) and the
  one or two lines you will change in existing files (mounting the route, loading the client);
- the guards from step 5, and how long a run takes.

Wait for "ok" or questions. Adjust, then build.

## 3. Build the locator

Follow [references/locator.md](references/locator.md) — signature, input caps, corpus, scoring,
rivals, confidence, handler search, the clipboard line and the HTTP contract. Keep the numbers it
gives as the starting point; they were tuned on a real 20k-line no-build app against measured
cases. Change a number only when a case you built shows why.

## 4. Build the client

Follow [references/client.md](references/client.md) — the mode, the button that stays reachable
over modals, capture-phase interception, the signature, the clipboard with its http fallback,
the plaque, and the self-off timer.

## 5. Build the guards — the feature is not done without them

Follow [references/guards.md](references/guards.md):

1. a **case set built from the live source** (find a line by regex, build the signature from it)
   — one case per ranking rule, plus a negative case;
2. an **autotest** that runs those cases against the locator, plus the input-abuse checks;
3. a **mutation probe**: copies of the locator each with one rule broken; every break must turn
   some case red. A probe that lets a break pass means a rule nobody guards — add a case;
4. **prove red once**: break one expectation by hand, watch the autotest fail, revert;
5. if a headless browser is already in the project, the end-to-end probe (mode on → tap → the
   app made zero requests of its own → the clipboard holds an address → Esc turns the mode off).

## 6. Prove it on the real screen

Run the app, turn the mode on (yourself through a headless browser if the project has one,
otherwise ask the person to tap and paste you the lines), tap at least six different kinds of element (a button with an id,
an item in a repeated list, something inside a modal, a plain text label, an icon, a container).
For each, open the returned line and check it is where that element is built. Report the result
as numbers — "5 of 6 exact, 1 came back `(?)` with the right line second" — never "it works".
A confident wrong answer is a bug to fix before you finish: it costs the person a whole session,
while an honest `(?)` costs ten seconds.

## 7. Hand it over

Tell the person, in a few plain lines: where the button is and its shortcut, what the line on
the clipboard means, what `(?)` means, that the mode switches itself off after three minutes idle,
how to run the guards, and the known limits you saw (references/locator.md, § limits). Put the
same in a short doc next to the code, written for the next session that touches it.

## Rules that hold through every step

- **No confident lies.** Below the confidence gap the answer ships as `(?)` with alternatives.
  Every change to ranking is checked against the case set and the mutation probe.
- **The corpus is only what the browser already downloads**, or templates whose line numbers
  reveal nothing a visitor cannot already see. Never index server logic, config or secrets.
- **Keep the tool's own files out of its corpus.** Their comments name other modules' classes
  and container ids, and pull answers into the tool itself.
- **The tool must not look like a broken app.** It is loud while on (a frame around the screen, a
  filled button), never survives a reload, and switches itself off when forgotten.
- **Write in the project's idiom** — its module style, naming, comment density and UI language.
  Nothing about tap2line should look bolted on.
