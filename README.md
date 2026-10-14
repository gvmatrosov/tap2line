# tap2line

Stop paying Claude Code to search for code you can point at.

Tap any element in your running app and get the source line that builds it:

```
src/screens/Cart.kt:118 · handler :131
```

It lands on your clipboard, and in a file Claude can read, so you either paste it or just say
"look at my last tap". The session starts at the line instead of searching for "the buy button
on the cart screen".

## Any platform

Web (plain JS, server templates, React, Vue, Svelte…), Electron, React Native, Flutter, Android
(Views and Compose), iOS (UIKit and SwiftUI), Qt, WPF, WinForms — and any other UI toolkit that
offers three things: a way to see a tap before the app does, a tree of elements under a point,
and something on each element that also appears in the source.

## What it is

A Claude Code plugin with one skill. **It ships no code.** Claude reads your project and builds
the tool inside it — in your language, your module style, your build — and then builds the tests
that keep it honest. You own every line; uninstalling the plugin removes nothing from your
project.

## Install

```
/plugin marketplace add gvmatrosov/tashi-solo
/plugin install tap2line@tashi-solo
```

Then ask Claude: *"set up tap2line"* (or run `/tap2line:setup`).

## How it works

- **Two halves.** The *tapper* lives in your debug build: a toggle switches a mode on, and while
  it is on a tap presses nothing — it reads what the UI toolkit knows about the element (ids,
  test tags, accessibility ids, type names, text, its ancestors, the current screen). The
  *locator* runs on your machine next to the source and turns that into an address.
- **Exact where the platform knows, ranked where it does not.** React, Vue, Svelte and Flutter
  debug builds can name an element's source position; tap2line uses it. Everywhere else the
  locator scores the lines of your UI source against the element at the moment you tap — nothing
  is stored or written into your code by hand, so nothing rots when a line is inserted above.
- **No confident lies.** When two places in the code could have built the element, the answer
  comes back as `(?)` with the alternatives. A wrong confident line costs a whole session; an
  honest "two candidates" costs ten seconds.
- **Debug only, developer only.** The tapper is not in release builds; the locator listens on
  your machine only (a phone reaches it through `adb reverse`, the simulator, or a one-time
  token on your Wi-Fi).

## The tests it builds

The feature is not done until these are green:

- a case per ranking rule, **built from your live source** (rename a button — the case rebuilds
  itself instead of failing);
- an autotest of the locator, including abuse checks (a 9000-name signature must not freeze it;
  junk input must not crash it) and the gate;
- a **mutation probe**: copies of the locator with one rule broken each — every break must turn
  some case red. It exists because the first version's tests survived six of six breaks.

## Lock or request

The skill is a request: Claude follows it like an instruction, nothing enforces it. The tests it
builds are the lock — they run in your project and fail when the inspector would lie.

## Status

Verified end to end on web projects without a build step, in two clean Claude sessions that had
only this skill: 33 taps on real elements, zero confident wrong answers (28 exact, 4 honest `(?)`
with the right line listed, 1 honest "not found"); every guard green and proven able to go red.
Other platforms are built from the same method and a per-platform guide; they have not had such a
run yet — reports welcome.

## Origin

Extracted from a real no-build web platform (~20k lines of hand-written JS), where it was measured
on 60 real signatures across 8 screens: after the review, the confident-lie count went from four
to zero.

## License

MIT
