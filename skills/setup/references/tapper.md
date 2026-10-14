# The tapper — a mode, not a gesture of its own

What every tapper must do, on any platform. *How* on each platform — the interception hook, the
toggle, the signature fields, the clipboard call — is in platforms.md. Prefix everything the
tapper adds (`t2l`, `T2l…`) and keep its own elements out of the signature.

## Boot

In a debug build only, ask the locator's `ping`. Anything but 200 → do nothing at all, show
nothing. Otherwise show the toggle. No flags in the app beyond the platform's own debug switch:
the locator's answer is the only on/off.

## Why a mode

Right-click, long-press and double-tap usually belong to the app already, and phones have no
right button. A mode avoids both problems: while it is on, a plain tap is a pick. It also gives the
person one clear state: on means the app is a specimen, off means the app is the app.

## The toggle

- A small floating button the app's own layers cannot cover: it must stay reachable while a
  modal, sheet, drawer or dialog is open. If it lived inside the header or a screen, it would
  vanish exactly over the modal layer — and the mode cannot be switched on beforehand, because in
  the mode the modal will not open. Put it in the topmost layer the platform offers (platforms.md).
- Where a keyboard exists, a shortcut too (Alt+I unless taken).
- It must not cover anything important; if it overlaps a corner of the app's bar, the app's bar
  gets matching padding while the tool is active.

## The mode

While on:

- **every touch/click/press reaches the tapper first and the app never sees it** — including
  touch-down/pointer-down, where drag, swipe and carousels start and would move before a click
  could be cancelled; also long-press and context-menu events. Nothing in the app may fire;
- hovering (where there is a pointer) frames the element under it with a small label of its type
  and name;
- `Esc` / the toggle switches off. Handle `Esc` before the "is this the tapper's own element"
  check: after a click on the toggle, focus stays on it, and Esc must still work;
- the screen is visibly different (a coloured frame around it, a filled toggle);
- it switches **itself off after 3 minutes without a pick**, and says so. A forgotten mode looks
  exactly like a broken app — nothing responds — the worst failure a diagnostic tool can have;
- it does **not** survive a reload or restart. Never persist it.

Switching off removes every hook and hides the frame **and** the plaque.

## The pick

1. Find the element under the tap — the deepest one the toolkit hit-tests to (platforms.md).
2. If the platform knows its exact source position (platforms.md, "exact source"), put it in
   `exact`. Build the signature anyway — the locator needs it for the handler and the record.
3. POST the signature to `locate`.
4. Put `line` on this device's clipboard.
5. Show the plaque: the line, coloured by result (exact/high · low · not found), and a second line
   when useful — "low confidence — check the list", "also used in N places".

## The signature

```
path:   from the tapped element up, at most 6 nodes, stopping at the app's root
        (above it is the shell shared by every screen — only noise).
node:   type, id, names, attrs, text (fields per platform in platforms.md)
screen: the current screen's name (platforms.md).
```

If the project has another tool that would want a source address for an element (a crawler that
files complaints), export the signature function — one signature, one coordinate system.

## The clipboard

Use the platform's clipboard API (platforms.md). If it fails, the plaque shows the line selected
in a read-only text field — losing the address silently is worse than asking for a manual copy.
The locator has already written it to `.tap2line/last.txt`, so "look at my last tap" works
either way. If the project already has a copy helper, use it.

## The plaque

Bottom centre, above everything. Hides after ~9 seconds, on tap, and when the mode switches off —
a plaque lingering over content after the mode ended gets in the way of real work. Set its text
as plain text, never as markup: the signature contains page text.
