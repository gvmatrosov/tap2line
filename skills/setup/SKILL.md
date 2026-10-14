---
name: setup
description: Build a tap-to-source-line inspector into this app, whatever it runs on — web (any stack), Android, iOS, Flutter, React Native, desktop (Electron, Qt, WPF…). A debug-only toggle in the running app; while it is on, tapping any element puts the file and line that builds it on the clipboard (`app/src/main/res/layout/cart.xml:42`, `src/Cart.tsx:118 · handler :131`) and in a file Claude can read, ready for Claude Code. Use when the person asks for a code inspector, "tap an element → find its code", "where is this button in the source", tap2line, or keeps describing on-screen things in words so every session searches for them again.
---

# tap2line — tap an element, get its source line

You are going to build, inside this person's project, a small debug-only tool with two halves:

- **the tapper** — lives in the running app. A toggle switches a mode on; while it is on, a tap
  presses nothing. It reads the tapped element's **signature** (whatever the UI toolkit knows about
  it: ids, type or class names, text, test tags, accessibility ids, its ancestors, the current
  screen) and sends it to the locator;
- **the locator** — runs on the developer's machine, next to the source. It answers with one
  address, or honestly with `(?)` and the alternatives.

The answer is one line — `src/screens/Cart.kt:118 · handler :131` — put on the clipboard of the
device that tapped **and** appended to `.tap2line/last.txt` in the project, so the person can
either paste it or just say "look at my last tap".

**The address is never kept by hand.** It comes either from the platform itself (a debug build
that knows the source position of each element — see platforms.md) or is computed at request
time from the live source. Nothing is written into the code by hand, so nothing rots when a line
is inserted above. Do not trade this away for a lookup table or hand-written `file:line` tags.

The same tool works on every platform; what changes is where the signature comes from, how the tap
is intercepted, and how the device reaches the locator. All of that is in
[references/platforms.md](references/platforms.md).

This skill is a request, not a lock: nothing checks that you followed it. It builds the tool and
nothing more — no test suite is left in the project; keeping the tool honest as the project grows
is the person's call. What makes *this* build true is step 5: real taps, checked by eye.

## 1. Look at the project — detect first, ask only what cannot be seen

Find out, by reading, not asking:

1. **The platform and UI toolkit** — web (which framework, if any; with or without a dev build),
   Android (Views/XML or Compose), iOS (UIKit or SwiftUI), Flutter, React Native, desktop. A
   project can have several (a web app plus a mobile app) — then build one locator and one tapper
   per UI, and say so in the plan.
2. **Whether the platform can name the source itself** in a debug build (platforms.md, "exact
   source"). Where it can, that is the answer; the ranking locator covers what it cannot.
3. **Where elements are born** — the files and the idioms that create them (`class="x"` in a
   template, `android:id="@+id/x"`, `Modifier.testTag("x")`, `.accessibilityIdentifier("x")`,
   `Key('x')`, `testID="x"`, `x:Name="x"`, `setObjectName("x")`). That set of files is the
   **corpus**; those idioms are the **birth patterns**. Not in the corpus: dependencies, generated
   code, build output, assets.
4. **Screens** — what names the current screen at runtime (a container id, an Activity/Fragment,
   a view controller, a route name). The screen name is the strongest file filter there is.
5. **Text tables** — if visible text comes from string resources or translation files, the
   locator resolves text → key → the line that uses the key.
6. **How a debug build is told apart** (`__DEV__`, `BuildConfig.DEBUG`, `#if DEBUG`,
   `kDebugMode`, a dev flag on the server) — the tapper must not exist in a release build.
7. **How the running app reaches the developer's machine** — same machine (web, desktop,
   simulators), USB (`adb reverse`), or the LAN (a physical phone).
8. **Taps already taken** — right-click menus, long-press, drag starting on touch-down, a shortcut
   that would clash with the toggle.
9. **A UI driver** already in the project (Playwright, Espresso, XCUITest, Flutter integration
   tests…) — only to drive the taps in step 5 yourself; you add nothing to its suites.

## 2. Show the plan and wait for an ok

Before writing code, give the person a short plan in plain words:

- the platform(s), and whether the answer will be exact (from the platform) or ranked;
- the corpus and the birth patterns, and what is left out and why;
- where the locator runs and how the app reaches it;
- **the gate**: debug builds only, and who else could reach the locator (references/locator.md,
  § gate). Say plainly if a physical phone needs it reachable on the LAN;
- the files you will add and the few lines you will change in existing files.

Wait for "ok" or questions. Adjust, then build.

## 3. Build the locator

Follow [references/locator.md](references/locator.md) — contract, gate, input caps, corpus,
scoring, rivals, confidence, handler search, the output line. The numbers there are the tuned
starting point (measured on a real 20k-line app); change one only when a case you built shows why.

## 4. Build the tapper

Follow [references/tapper.md](references/tapper.md) for what every tapper must do (a loud,
temporary mode that intercepts taps before the app sees them, a toggle reachable over modals, the
signature, the clipboard) and [references/platforms.md](references/platforms.md) for how to do
it on this platform.

## 5. Prove it on the real screen

Run the app, switch the mode on, tap at least six different kinds of element (one with an id,
an item in a repeated list, something inside a modal or sheet, a plain text label, an icon, a
container). Do it yourself through the project's UI-test driver if there is one; otherwise ask the
person to tap and then read `.tap2line/last.txt`. Close whatever is open
(a modal, a sheet) before the next tap, or the tap lands on its overlay and the overlay's line is
the right answer to the wrong question. For each, open the returned line and check that it builds
that element. Report numbers: "5 of 6 exact, 1 came back `(?)` with the right line
second". A confident wrong answer is a bug to fix before you finish: it costs the person a whole
session, while an honest `(?)` costs ten seconds.

## 6. Hand it over

Tell the person, in a few plain lines: how to switch the mode on, what the line means, what
`(?)` means, "look at my last tap" as an alternative to pasting, that the mode switches itself off
after three idle minutes, and the known limits you saw. Put the same in a
short doc next to the code, for the next session that touches it. Add `.tap2line/` to the
project's ignore file.

## Rules that hold through every step

- **No confident lies.** Below the confidence gap the answer ships as `(?)` with alternatives.
- **Nothing in a release build.** The tapper is compiled out or never loaded outside debug; the
  locator never runs in production.
- **The locator answers only the developer** and returns positions, never file contents. Its
  corpus is UI source (including server code that emits markup), never config or secrets.
- **Keep the tool's own files out of its corpus.** Their comments name other modules' classes and
  pull answers into the tool itself.
- **The tool must not look like a broken app.** It is loud while on, never survives a restart,
  and switches itself off when forgotten.
- **Write in the project's idiom** — its language, module style, naming, comment density and UI
  language. Nothing about tap2line should look bolted on.
