# Guards — the case set, the autotest, the mutation probe

A ranking heuristic fails quietly: it keeps answering, just the wrong line. So the tool ships
with guards that prove each ranking rule does its job, and with a probe that proves the guards
themselves can fail.

## 1. The case set — one module, two users

One file exporting `buildCases()`, used by both the autotest and the mutation probe. Two lists
would drift, and the probe would start "confirming" tests that no longer exist.

**Every case is built from the live source**, not pinned to a named button: find a line with a
regex, build the signature from what that line and its surroundings contain, and expect that
line. A renamed control then rebuilds its case instead of turning it red for nothing. If a regex
finds nothing in this codebase, skip that case with a printed note — and if that leaves a rule
without any case, say so in the hand-over.

**Every case carries a real ancestor chain** (classes of the enclosing elements in the same
template). A bare element is a laboratory signature; it stops catching what breaks in the field.

One case per rule, each named after the rule it guards:

| # | case | expectation |
|---|---|---|
| 1 | element with a unique id | exact line, confidence high |
| 2 | hand-repeated identical markup (several tiles written out one after another) | the tapped tile's own line, not the first one; with no distinguishing ancestor — `(?)` |
| 3 | same signature in two different screen containers | different files for the two `screen` values |
| 4 | a token with both a birth line and a `querySelector` line | the birth line |
| 5 | a unique id next to a common class used in many places | confidence stays high |
| 6 | a rare class next to a common one | the rare class's line wins, confidence high |
| 7 | handler wired near the birth line | `handler` is that line |
| 8 | **negative**: tokens that exist nowhere in the corpus | `found: false`, nothing invented |

Build what this codebase can supply; if it has no repeated tiles, construct none — a case
must come from the real source to mean anything.

## 2. The autotest

Run every case through the locator over HTTP against the running dev server (that also checks
the route, the gate and the JSON), compare file, line and confidence, print one line per case,
exit non-zero on any failure. Add:

- **abuse**: a signature with 9000 classes must answer in well under a second (the caps work);
  junk types (`cls: "x"`, `path: {}`, `data: []`) must give 400 or a normal answer, never 500;
- **the gate**: without the owner's session (or with the dev flag off) both routes answer 404.

Wire it into the project's existing test command. If the project has none, add one script and
tell the person how to run it.

## 3. The mutation probe — the guard on the guard

The first version of this tool had an autotest that survived six of six deliberate breaks: it
proved only that grep is alive and HTTP answers. Hence the probe:

1. copy the locator source into a temp directory;
2. for each rule, make one copy with exactly that rule broken by a literal string replace;
3. load each copy, run the full case set against it, and require **at least one red case per
   break**;
4. run the unbroken copy first as a control: it must be all green, or there is nothing to measure.

The breaks, at minimum:

| break | replace |
|---|---|
| birth bonus | `+4` → `+0` |
| birth mark (rival rule) | `birth = true` → `birth = false` |
| screen filter | `+5` → `+0` |
| rarity | `rarity(n)` → always 1 |
| ancestor distance scale | `6 / (1 + d)` → `0` |
| rival must share the id | the same-id check → always true |
| confidence gap | `× 1.6` → `× 1.0` |
| handler search | `≤ 150` → `≤ 0` |

The replace strings must match the code exactly; if a replace finds nothing, the probe fails
loudly (otherwise a refactor silently turns a break into a no-op that "passes"). Keep the probe
next to the tests, run it after any change to ranking, and print a table: break → which case went
red. A break no case catches is a rule nobody guards — add a case, do not delete the break.

## 4. Prove red once

Before trusting a green autotest: invert one expectation by hand (expect the wrong line), run,
watch it fail, revert. A test that cannot fail is false comfort.

## 5. End-to-end browser probe (only if a headless browser is already in the project)

Log in the way the project's own browser tests do (never a door added for the test), open a
screen, then: the button exists → mode on → tap a known element → **the app made zero network
requests of its own** besides `locate` → the clipboard holds an address (grant clipboard
permission, or read the plaque) → `Esc` → mode off, no frame, no plaque, a tap presses normally
again. If the login goes stale, the probe must fail, not silently run as anonymous and prove
nothing.

## No data sensor

The tool writes nothing and keeps no state at rest, so there is no data invariant to guard.
