# The client — a mode, not a right-click

Plain JS in the project's module style, plus a small CSS file. Prefix every id and class it adds
(`t2l-`) and leave those prefixed classes out of the signature.

## Boot

On load, ask `ping`. Anything but 200 → do nothing at all, draw nothing. Otherwise load the CSS
and add the button. No flags, no config in the client: the server's answer is the only switch.

## The button

- One small fixed button appended to `body` — **not inside the header**. A sticky header sits
  below every modal and drawer; from inside it the button is unreachable while a modal is open,
  and the mode cannot be switched on beforehand because in the mode the modal will not open.
  That would cut the whole modal layer off. Give it a z-index above everything the app uses.
- Place it where it covers nothing important; if it sits over the header's corner, give the
  header a matching `padding-right` (only when the tool is active, via a class on `<html>`).
- A keyboard toggle (Alt+I unless the app uses it).

## The mode

While on:

- listen in the **capture phase** on `document` for `click`, `pointerdown`, `contextmenu` and
  `keydown`. `pointerdown` matters: drag and carousels start there and would move before the click
  could be cancelled. In each: `preventDefault`, `stopPropagation`, `stopImmediatePropagation` —
  except on the tool's own button and plaque. Nothing in the app may fire;
- `pointerover` draws a frame around the element under the pointer with a small `tag.class` tip;
- `Esc` switches off;
- the screen is visibly different (a coloured frame around the viewport, a filled button);
- it switches **itself off after 3 minutes without a pick**, and says so. A forgotten mode looks
  exactly like a broken app — nothing responds — the worst failure a diagnostic tool can have;
- it does **not** survive a reload. Never persist it.

Switching off removes every listener and hides the frame **and** the plaque.

## The pick

1. Build the signature (below), POST it to `locate`.
2. Put `clip` on the clipboard (below).
3. Show the plaque: the clip line, coloured by result (high / low / not found), and a second line
   when useful — "low confidence — check the list", "also used in N places".

## The signature

```
path: walk from the tapped element up, at most 6 nodes, stop at the app root (#app/#view/main)
      — above it is the shell shared by every screen; it only adds noise.
node: { tag, id, cls: [...classList minus t2l-*], data: {...dataset},
        text: textContent with whitespace collapsed, first 60 chars }
screen: the nearest ancestor (or self) with an id, not counting the app root.
```

Export the signature function if anything else in the project could reuse it (a crawler that
wants to attach a source address to a complaint) — one signature, one coordinate system.

## The clipboard

`navigator.clipboard` exists only in a secure context (https or localhost). Dev servers on a LAN
IP over plain http do not have it. So, two roads:

1. `navigator.clipboard.writeText` when `window.isSecureContext`;
2. otherwise a hidden off-screen `<textarea>` (readonly, `position:fixed; top:-1000px`), select
   it, `document.execCommand('copy')`, remove it.

If both fail, the plaque shows the line in a selected read-only textarea — losing the address
silently is worse than asking for a manual copy. If the project already has a copy helper, use
it instead of writing a second one.

## The plaque

Fixed near the bottom centre, above everything. Hides after ~9 seconds, on tap, and when the mode
switches off — a plaque lingering over content after the mode ended gets in the way of real work.
Text via `textContent`, never `innerHTML` — the signature contains page text.
