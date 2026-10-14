# tap2line

Tap any element in your running web app and get the source line that builds it on your
clipboard:

```
public/cart.js:212 · handler :240
```

Paste it into Claude Code, and the session starts at the line instead of searching for "the buy
button on the cart screen".

## What it is

A Claude Code plugin with one skill. **It ships no code.** Claude reads your project and builds
the inspector inside it, in your language, your module style and your server's routing — then
builds the tests that keep it honest. You own every line; uninstalling the plugin removes nothing
from your project.

Made for the projects other inspectors skip: markup born in hand-written JS template strings, DOM
built by hand, plain HTML, server templates (EJS, Jinja, Blade, ERB, PHP…). No build step needed.
If your UI is React/Vue/Svelte with a dev build, a compiler-based tool gives exact answers; the
skill says so first and lets you choose.

## Install

```
/plugin marketplace add gvmatrosov/tashi-solo
/plugin install tap2line@tashi-solo
```

Then ask Claude: *"set up tap2line"* (or run `/tap2line:setup`).

## How it works

- **A mode, not a right-click.** A small button switches the mode on. While it is on, a tap
  presses nothing — it reads the element's signature (tag, id, classes, `data-*`, text, five
  ancestors, the screen container) and asks the locator. Works on a phone. Esc or three idle
  minutes switch it off.
- **Computed, not stamped.** The locator scores every line of your client source against the
  signature at request time. Nothing is stored in the markup, so nothing rots when a line is
  inserted above.
- **No confident lies.** When two places in the code could have built the element, the answer
  comes back as `(?)` with the alternatives. A wrong confident line costs a whole session; an
  honest "two candidates" costs ten seconds.
- **Dev only.** The routes exist only in development, and answer only you if your dev server is
  reachable by others. They index only files your browser already downloads.

## The tests it builds

The feature is not done until these are green:

- a case per ranking rule, **built from your live source** (rename a button — the case
  rebuilds itself instead of failing);
- an autotest over HTTP, including abuse checks (a 9000-class signature must not freeze the
  server; junk input must not 500);
- a **mutation probe**: copies of the locator with one rule broken each — every break must turn
  some case red. It exists because the first version's tests survived six of six breaks.

## Lock or request

The skill is a request: Claude follows it like an instruction, nothing enforces it. The tests it
builds are the lock — they run in your project and fail when the inspector would lie.

## Origin

Extracted from a real no-build school platform (~20k lines of hand-written JS), where it was
measured on 60 real signatures across 8 screens: after the review, the confident-lie count went
from four to zero.

## License

MIT
