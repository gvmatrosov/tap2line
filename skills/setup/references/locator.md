# The locator — signature in, address out

Write it in the project's server language and idiom. The algorithm below is language-neutral;
numbers are the tuned starting point.

## Contract

```
GET  <dev prefix>/ping     → 200 {"ok": true, "files": <corpus size>}   — or 404
POST <dev prefix>/locate   → 200 {found, confidence, best, others, signature, clip}
                             400 when the signature has no path
                             404 when the gate says no
```

The client draws its button only if `ping` answers 200. So "not mounted" and "not allowed" both
look the same from outside — a 404 — and the client has one rule: no answer, no button.

Request body (the signature):

```json
{
  "screen": "cart-view",
  "path": [
    { "tag": "button", "id": null, "cls": ["btn", "btn-buy"], "data": { "sku": "A12" }, "text": "Buy now" },
    { "tag": "div", "id": null, "cls": ["cart-row"], "data": {}, "text": "…" }
  ]
}
```

`path[0]` is the tapped element, then its ancestors, nearest first. `screen` is the nearest
ancestor id that is not the app root.

Response:

```json
{
  "found": true,
  "confidence": "high",
  "best": { "file": "public/cart.js", "line": 212, "score": 23.5, "handler": 240 },
  "others": [ { "file": "public/promo.js", "line": 88, "score": 9.1 } ],
  "signature": "button.btn.btn-buy · inside #cart-view · text «Buy now»",
  "clip": "public/cart.js:212 · handler :240"
}
```

`file` is relative to the repository root, so it can be opened as written.

## Gate

Two gates, both answering 404:

1. **absence** — mount the routes only in development (the project's own dev flag). In
   production they do not exist;
2. **who** — with no login in the project, answer only loopback callers (`127.0.0.1`, `::1`)
   and refuse anything that arrived through a proxy (a forwarding header present). Tell the person
   the price: a phone on the LAN cannot use it then; for a phone, gate on a login instead.
   If the dev server is reachable by anyone but the developer (a shared dev box, a
   staging site, a server where dev and real users are the same process), let only the project's
   owner/admin through, reusing the project's existing auth check. If the project impersonates
   users ("view as"), check the real actor, not the impersonated one — the inspector is used most
   on exactly those screens.

## Input caps — normalise before anything else

Every token is one pass over the whole corpus; in a single-threaded server a huge signature
freezes every user. Cap and type-coerce in the function itself (the tests call it directly too):

- at most 6 nodes in `path`; per node at most 12 classes, 8 `data-*` entries;
- every token at most 64 chars; `text` at most 120; `tag` only `[\w-]`, at most 20;
- `data-*` keys only `^[\w-]{1,40}$` with string values;
- wrong types (a string where an array belongs) are dropped, never a 500.

## Corpus

The files that build what the browser shows (SKILL.md step 1), read as lines. Cache for ~2
seconds: long enough to survive a burst of taps, short enough that an edit is seen on the next
refresh. Exclude the tool's own files by name.

## Tokens and weights

From each node take: its id, each class longer than one char, each `data-<key>`, and each
`data-*` value that is 3–39 chars and not all digits (record ids never appear in code;
`data-tab="orders"` often does).

| token | tapped element | ancestor |
|---|---|---|
| id | 10 | 3 |
| class | 6 | 2 |
| `data-<key>` | 4 | 1.5 |
| `data-*` value | 3 | 1 |

Match tokens as whole names with a boundary that treats `-` as part of the name — `hw-card` must
not match inside `hw-card-x`, and `card` must not match inside `hw-card`. Standard `\b` gets this
wrong. Use `(?<![\w-])TOKEN(?![\w-])` (escape the token).

## Candidates

Only the **tapped element's own** tokens create candidates; ancestors only add points to
candidates that already exist. For each own token: find every line containing it; skip the token
if it hits more than **60** lines (it means nothing); otherwise every hit line gets
`weight × rarity(hits)`:

```
rarity(n) = 1.5 if n ≤ 2;  1 if n ≤ 6;  0.5 if n ≤ 20;  0.2 otherwise
```

A rare exact name must outweigh a common utility class that merely appears many times.

No candidates at all → `found: false`. Never invent an answer.

## Bonuses on each candidate line

| signal | points | why |
|---|---|---|
| file is the **screen file** | +5 | the container id is written in the module that draws it |
| the line opens a tag of the element's own tag (`<button`) | +3 | |
| **birth line** | +4, and mark `birth` | the line builds the element rather than refers to it |
| the element's visible text appears literally | +4 | only when the text is 3–40 chars and contains a letter; text from data is not in the code |
| ancestors | up to +16 total | see below |

**Screen file** = the corpus file with the most lines matching `id=["']<screen>["']`. Take the
most, not the first: a neighbour's comment can mention the id once.

**Birth** (§ birth) — a line where a matched token is written as the element's class or id:
`class="… x …"`, `id="x"`, `className = '… x …'`, `classList.add('x'`. Extend the pattern for
what this codebase writes (`setAttribute('class'`, `:class=`, a template helper). A
`querySelector('.x')` line is a reference, not a birth.

**Ancestors** — for every ancestor token that hits 1–60 lines, looking only at hits in the
candidate's own file:

- +2 if it occurs in the file at all;
- +3 more if one occurrence is within 40 lines;
- `+ 6 / (1 + d)` where `d` is the distance to the nearest occurrence, **doubled when the
  candidate is above the ancestor** — in a template the child follows its parent's opening tag.
  This smooth scale is what separates five hand-written identical tiles in a row; a flat "nearby"
  bonus gives all five the same score and the first one wins.

Cap the ancestor sum at 16.

## Merge, rivals, confidence

Sort by score. Merge lines within 3 lines of a better one in the same file — a template's
neighbouring lines are the same control.

The best candidate is `best`. A **rival** is a candidate that would be a *different answer*:

- if `best` is a birth line, only another birth line can rival it — but any other birth line,
  at any distance (hand-repeated markup is exactly the case to catch);
- if `best` matched the element's **own id**, a rival must match that id too — lines sharing only
  a decorative class are different elements, not competing answers;
- likewise, if `best` matched one of the element's **rare** own tokens (2 hits or fewer in the
  corpus), a rival must match one of those rare tokens too. A sort button sharing only `.btn` with
  "Add to cart" is another element, not a competing answer;
- otherwise, a rival is any candidate in another file, or more than 60 lines away in the same file.

```
confidence = "high"  if (no rival  or  best.score ≥ rival.score × 1.6)  and  best.score ≥ 8
             "low"   otherwise
```

Confidence is about the **gap**, not the score. Twenty identical cards on screen built by one
function are one address — repetition on screen is not ambiguity in code. Real ambiguity is real
copy-paste in the source, and saying so is often the most useful finding.

`others` = up to 3 next candidates that are rivals or far enough away — shown even when
confidence is high, so the person sees that the same class lives elsewhere.

## Handler

In `best`'s file, the nearest line (other than `best` itself, within 150 lines) that contains one
of `best`'s matched tokens **and** one of: `addEventListener`, `onclick`, `.on<event> =`,
`querySelector`, `closest(`, `matches(` (extend for the project's event idiom). That is where the
element's behaviour is wired. Report it as `handler`.

## The clipboard line

```
found, high:   <file>:<line>[ · handler :<h>]
found, low:    <file>:<line> (?)[ · handler :<h>] · also: <f>:<l> · <f>:<l> · sig: <signature>
not found:     not found in <corpus dirs> · sig: <signature>
```

`signature` reads like `button#buy.btn.btn-buy · inside #cart-view · text «Buy now»`. Match the
labels to the project's UI language if it is not English; keep the structure.

## Known limits — say them in the hand-over doc

- A sub-screen drawn into **another module's container**, reusing that module's classes: the
  screen filter points at the wrong module, the right line comes back among the alternatives.
- **Classes assembled in a variable** (`` const cls = `row${x}` `` then `class="${cls}"`): no
  literal birth line, the answer lands near the render function, not on it.
- **Elements with no literal of their own** (a bare `<div>`/`<span>`): "not found", honestly.
