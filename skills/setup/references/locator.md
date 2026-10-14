# The locator — signature in, address out

It runs on the developer's machine, next to the source, on every platform. Write it in the
project's own server/tooling language when there is one (a dev-only route in a web server), or as
a standalone zero-dependency script in whatever runtime the machine surely has (Node, Python).
The algorithm is plain string scanning; the numbers below are the tuned starting point.

## Contract

```
GET  <base>/ping     → 200 {"ok": true, "files": <corpus size>}   — or 404
POST <base>/locate   → 200 {found, confidence, best, others, signature, line}
                       400 when the signature has no path
                       404 when the gate says no
```

The tapper turns its toggle on only if `ping` answers 200. "Not running" and "not allowed" look
the same from outside, so the tapper has one rule: no answer, no toggle.

If the platform already gave an exact source position (platforms.md), the tapper sends it as
`exact: {file, line}`; the locator checks the file exists in the corpus, takes it as `best` with
confidence `exact`, and still searches for the handler.

The signature, platform-neutral:

```json
{
  "screen": "CartFragment",
  "exact": null,
  "path": [
    { "type": "button", "id": "buy_now", "names": ["btn", "btn-buy"],
      "attrs": { "sku": "A12" }, "text": "Buy now" },
    { "type": "LinearLayout", "id": "cart_footer", "names": [], "attrs": {}, "text": "…" }
  ]
}
```

`path[0]` is the tapped element, then its ancestors, nearest first. What fills each field, per
platform, is in platforms.md:

- `id` — the one identifier the toolkit treats as unique: DOM id, Android resource entry name,
  accessibility identifier, test tag / testID, Flutter `Key`, `objectName`, `x:Name`;
- `names` — the type-ish names: CSS classes, a custom view/widget/component class name;
- `attrs` — the rest of the declared attributes worth searching (`data-*`, content description,
  style name);
- `type` — the tag or element kind (`button`, `TextView`, `Text`);
- `text` — the visible text, whitespace collapsed, first 60 chars;
- `screen` — the current screen: a container id, Activity/Fragment, view controller, route.

Response:

```json
{
  "found": true,
  "confidence": "high",
  "best": { "file": "app/src/main/res/layout/fragment_cart.xml", "line": 42, "score": 23.5, "handler": null },
  "others": [ { "file": "app/src/main/java/shop/CartFragment.kt", "line": 57, "score": 9.1 } ],
  "signature": "Button#buy_now · on CartFragment · text «Buy now»",
  "line": "app/src/main/res/layout/fragment_cart.xml:42 · also: CartFragment.kt:57"
}
```

`file` is relative to the repository root, so it opens as written. Every answer is also appended
to `.tap2line/last.txt` (keep the last ~20) with the time and the signature, so Claude can read it
without a paste.

## Gate

1. **Debug only** — the tapper exists only in debug builds (platforms.md has each platform's
   switch); a web server mounts the routes only under its dev flag.
2. **Only the developer** — the locator listens on loopback by default. That covers browsers on
   the same machine, desktop apps, iOS simulators, and Android via `adb reverse tcp:<port>
   tcp:<port>` (emulator or USB phone). A phone over Wi-Fi needs the LAN: then the locator prints a
   random token at start, the person enters it once in the tapper, and every request without it
   gets 404. If the locator sits inside a web dev server that others can reach, reuse the
   project's own owner/admin check instead; if the project impersonates users ("view as"), check
   the real actor. Refuse proxied requests (a forwarding header present) when the gate is
   loopback.

Tell the person the trade-off in the plan: loopback is the safest default, and a Wi-Fi phone
needs the token.

## Input caps — normalise before anything else

Each token is one pass over the corpus; a huge signature must not freeze the machine (or, inside a
web server, every user). Cap and type-coerce inside the function itself (tests call it directly):

- at most 6 nodes in `path`; per node at most 12 `names`, 8 `attrs`;
- every token at most 64 chars; `text` at most 120; `type` only `[\w.-]`, at most 40;
- `attrs` keys only `^[\w-]{1,40}$` with string values;
- wrong types are dropped, never a crash or a 500.

## Corpus

The files where the UI is born (SKILL.md step 1), read as lines, plus the text tables (string
resources, translation files). That includes server code that emits markup as strings (a
`renderRow()` helper in a route file) — otherwise its elements come back "not found". The locator
answers only the developer and returns positions, never file contents, so this reveals nothing;
still leave out config, secrets and anything that is not UI. Cache for ~2 seconds: survives a burst of taps, sees an edit on
the next one. Exclude the tool's own files by name.

## Tokens and weights

From each node: `id`; each name longer than one char; each attr key; each attr value that is
3–39 chars and not all digits (record ids never appear in code).

| token | tapped element | ancestor |
|---|---|---|
| id | 10 | 3 |
| name (class, custom type) | 6 | 2 |
| attr key | 4 | 1.5 |
| attr value | 3 | 1 |

Platform types that every element shares (`div`, `TextView`, `UIButton`, `Text`) are not tokens;
they only feed the "own type" bonus below. A custom type (`PriceTag`, `CartRowView`) is a name.

Match tokens as whole names with a boundary that keeps `-` and `_` inside the name — `hw-card`
must not match inside `hw-card-x`, and `card` must not match inside `hw-card`. Standard `\b` gets
this wrong. Use `(?<![\w-])TOKEN(?![\w-])` (escape the token).

## Candidates

Only the **tapped element's own** tokens create candidates; ancestors only add points to
candidates that already exist. For each own token: find every line containing it; skip the token
if it hits more than **60** lines (it means nothing); otherwise every hit line gets
`weight × rarity(hits)`:

```
rarity(n) = 1.5 if n ≤ 2;  1 if n ≤ 6;  0.5 if n ≤ 20;  0.2 otherwise
```

**Text through a table:** if the visible text is not a literal anywhere but is a value in a string
table (`<string name="buy_label">Buy now</string>`, `"buy": "Buy now"`), take the key and search
for it as an own token (weight 6) — `R.string.buy_label`, `@string/buy_label`, `t('buy')`.

No candidates at all → `found: false`. Never invent an answer.

## Bonuses on each candidate line

| signal | points | why |
|---|---|---|
| file is the **screen file** | +5 | the screen's name points at the module that draws it |
| the line opens/creates the element's own type (`<button`, `<Button`, `Button(`) | +3 | |
| **birth line** | +4, and mark `birth` | the line declares the element rather than refers to it |
| the visible text appears literally | +4 | only for 3–40 chars containing a letter; text from data is not in the code |
| ancestors | up to +16 total | see below |

**Screen file** — web: the corpus file with the most lines matching `id=["']<screen>["']` (the
most, not the first: a neighbour's comment can mention it once). Class-named screens
(`CartFragment`, `CartViewController`, a route `CartScreen`): the file that declares that class or
route, and its layout if it inflates one.

**Birth** — a line where a matched token is written as the element's identity in the platform's
own idiom: `class="… x …"`, `id="x"`, `className = '… x …'`, `classList.add('x'` (web);
`android:id="@+id/x"`, `Modifier.testTag("x")` (Android); `.accessibilityIdentifier("x")`,
`accessibilityIdentifier = "x"` (iOS); `Key('x')` / `ValueKey('x')` (Flutter); `testID="x"`,
`nativeID="x"` (React Native); `x:Name="x"`, `objectName: "x"`, `setObjectName("x")` (desktop).
Extend for what this codebase writes. A lookup (`querySelector('.x')`, `findViewById(R.id.x)`,
`find.byKey`) is a reference, not a birth.

**Ancestors** — for every ancestor token that hits 1–60 lines, looking only at hits in the
candidate's own file:

- +2 if it occurs in the file at all;
- +3 more if one occurrence is within 40 lines;
- `+ 6 / (1 + d)` where `d` is the distance to the nearest occurrence, **doubled when the
  candidate is above the ancestor** — in markup and in nested UI code the child follows its
  parent's opening. This smooth scale is what separates hand-written identical tiles in a row; a
  flat "nearby" bonus gives them all the same score and the first one wins.

Cap the flat part (the +2 and +3 bonuses) at 16 in total, so ancestors cannot outvote the
element's own tokens — but **never cap the distance scale**: it is the tie-breaker. A clean run on
five hand-written tiles showed a cap over the whole sum lets all five reach the same maximum, and
the section that tells them apart stops counting.

## Merge, rivals, confidence

Sort by score. Merge lines within 3 lines of a better one in the same file — neighbouring lines
of one declaration are the same element.

The best candidate is `best`. A **rival** is a candidate that would be a *different answer*:

- if `best` is a birth line, only another birth line can rival it — but any other birth line,
  at any distance (hand-repeated markup is exactly the case to catch);
- if `best` matched the element's **own id**, a rival must match that id too — lines sharing only
  a decorative name are different elements, not competing answers;
- likewise, if `best` matched any of the element's **rare** own tokens (2 hits or fewer in the
  corpus), a rival must match **every** rare token `best` matched — a line sharing one uncommon
  class but missing the other is a different element (a Sort button with `btn-ghost` is not a
  rival to a Details button with `btn-ghost btn-info`);
- otherwise, a rival is any candidate in another file, or more than 60 lines away in the same file.

```
confidence = "exact" if the platform gave the position
             "high"  if (no rival  or  best.score ≥ rival.score × 1.6)  and  best.score ≥ 8
             "low"   otherwise
```

Confidence is about the **gap**, not the score. Twenty identical cards on screen built by one
function are one address — repetition on screen is not ambiguity in code. Real ambiguity is real
copy-paste in the source, and saying so is often the most useful finding.

`others` = up to 3 next candidates that are rivals or far enough away — shown even when
confidence is high, so the person sees the same name lives elsewhere. A layout file and the code
that wires it (`fragment_cart.xml` and `CartFragment.kt`) are both useful: show the second as
`also:`.

## Handler

Near `best` (same file, within 150 lines; for a layout file, also the screen's code file), the
nearest line other than `best` that contains one of `best`'s matched tokens **and** a wiring
idiom: `addEventListener`, `onclick`, `.on<event> =`, `onClick`, `setOnClickListener`,
`addTarget(`, `@IBAction`, `onPressed:`, `onTap:`, `onPress=`, `clicked.connect`, `Click="…"`,
`querySelector`, `closest(`, `findViewById(` (extend for the project's idiom). Report it as
`handler`.

## The output line

```
exact / high:  <file>:<line>[ · handler :<h>][ · also: <f>:<l>]
low:           <file>:<line> (?)[ · handler :<h>] · also: <f>:<l> · <f>:<l> · sig: <signature>
not found:     not found in <corpus dirs> · sig: <signature>
```

`signature` reads like `Button#buy_now · on CartFragment · text «Buy now»`. Match the labels to
the project's UI language if it is not English; keep the structure.

## Known limits — say them in the hand-over doc

- A sub-screen drawn into **another module's container**, reusing its names: the screen filter
  points at the wrong module; the right line comes back among the alternatives.
- **Names assembled at runtime** (`` `row${x}` ``, `"item_" + i`): no literal birth line, the
  answer lands near the builder, not on it.
- **Elements with nothing of their own** — no id, no custom name, no literal text: "not found",
  honestly. On platforms with a thin runtime tree (SwiftUI, Compose without test tags) this is
  common; the fix is the platform's exact source where it exists, or an id/test tag on the
  element, which also helps UI tests.
