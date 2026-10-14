# Platforms — where the signature comes from, how the tap is caught, how the app reaches the locator

The locator (locator.md) and the tapper's behaviour (tapper.md) are the same everywhere. This file
fills in the platform-specific parts. **APIs named here move between versions** — check each one
against the SDK actually installed in the project before relying on it, prefer public API, and
when you fall back to an internal one, say so in the hand-over doc.

Every platform needs the same answers: **debug switch · toggle layer · interception · hit test ·
signature fields · screen name · exact source (if any) · birth patterns · clipboard · how the app
reaches the locator · UI-test driver.**

## Exact source — use it where the platform has it

Some toolchains already know, in a debug build, the source position of each element. That is the
best answer there is (confidence `exact`); the ranking locator covers the rest. A position stamped
by the **compiler** is fine — it is recomputed on every build, so it cannot rot; a position written
by hand is not.

| platform | where the exact position is |
|---|---|
| React (web, RN) ≤ 18 | dev builds keep `_debugSource {fileName, lineNumber}` on the fiber (DOM node key `__reactFiber$…`) |
| React 19+ | `_debugSource` is gone — add a dev-only JSX transform (Babel/SWC) that stamps `data-t2l="file:line"` on host elements |
| Vue 3 | a dev plugin that stamps positions (e.g. `vite-plugin-vue-inspector`'s `data-v-inspector`) if present, else a small compiler transform; `__vueParentComponent.type.__file` gives the file only |
| Svelte | dev builds attach `__svelte_meta` with the element's `loc` (file, line, column) — check its shape in the installed version |
| Flutter | widget creation locations (`--track-widget-creation`, on by default in debug), as used by the widget inspector |
| Jetpack Compose | the compiler's `sourceInformation` in debug builds (read by the layout inspector via ui-tooling) — heavy to reach; rank first, add this only if the person wants it |
| Everything else | none — ranked |

Where an exact source is used, still build the ranking locator: it finds the handler and covers
elements the compiler did not stamp.

## Web — no build, server templates, any framework

- **Debug switch:** the dev server's flag; load the tapper script only when it is on (or let it
  ask `ping` and stay silent on 404).
- **Toggle layer:** a fixed button appended to `body` with a z-index above every layer the app
  uses — not inside a sticky header, which sits under every modal.
- **Interception:** listeners in the **capture phase** on `document` for `click`, `pointerdown`,
  `contextmenu`, `keydown`; in each `preventDefault`, `stopPropagation`,
  `stopImmediatePropagation`, except on the tapper's own elements. `pointerover` draws the frame.
- **Hit test:** `event.target`.
- **Signature:** `type` = tag; `id`; `names` = classes (minus the tapper's `t2l-`); `attrs` =
  `dataset`; `text` = `textContent`. Stop the path at the app root (`#app`, `#view`, `main`).
  With a component framework, add the component names of the nearest components to `names`
  (from the dev tools hooks) — they name files.
- **Screen:** the nearest ancestor id that is not the app root; with a router, the route's
  component name.
- **Birth:** `class="…"`, `id="…"`, `className =`, `classList.add(`, `setAttribute('class'`;
  JSX `className="…"`; Vue `:class`/`class`; Svelte `class:x`; template engines' own syntax
  (`class="{{ … }} x"`).
- **Corpus:** the client source (JS/TS, components, `.html`) and server templates (EJS, Jinja,
  Blade, ERB, Go templates, PHP, Twig). Not `node_modules`, `*.min.js`, `dist/`.
- **Clipboard:** `navigator.clipboard.writeText` exists only in a secure context (https or
  localhost); on a LAN IP over plain http, use a hidden off-screen read-only `<textarea>`,
  `select()`, `document.execCommand('copy')`.
- **Reaching the locator:** a dev-only route in the dev server; with no server (static files),
  a standalone script on its own localhost port that allows CORS only from the app's dev origin,
  never `*` — and serves https if the app does, or the browser blocks it as mixed content.
- **UI-test driver:** Playwright, Cypress, Puppeteer.
- **Electron:** the renderer is the web case; the clipboard is also reachable through Electron's
  `clipboard` module.

## React Native

- **Debug switch:** `__DEV__`.
- **Toggle layer:** RN `Modal`s render in their own native window, above anything in the root
  view — so add a **dev-menu item** (`DevSettings.addMenuItem`) as the toggle that always works,
  plus a floating button in the root view for the common case.
- **Interception:** a root `View` with `onStartShouldSetResponderCapture={() => modeOn}` takes
  every touch before the app.
- **Hit test / exact source:** RN's own element inspector computes the element and its React
  source at a point (an internal API that changes between versions — check the installed one);
  with React ≤ 18 the fiber's `_debugSource` is the exact position.
- **Signature:** `id` = `testID` / `nativeID`; `names` = component names on the path; `text`;
  `attrs` = `accessibilityLabel`.
- **Screen:** the focused route's name from the navigation library.
- **Birth:** `testID="x"`, `nativeID="x"`, `<ComponentName`.
- **Clipboard:** the clipboard module the project already uses (e.g.
  `@react-native-clipboard/clipboard`); else rely on `.tap2line/last.txt`.
- **Reaching the locator:** the device already reaches the Metro host — run the locator on that
  host. Android: `adb reverse tcp:<port> tcp:<port>`, then `127.0.0.1`. iOS simulator:
  `localhost`. A physical phone over Wi-Fi: the host's LAN address plus the token (locator.md §
  gate).
- **UI-test driver:** Detox, Maestro.

## Flutter

- **Debug switch:** `kDebugMode`.
- **Toggle layer:** `MaterialApp`/`CupertinoApp`/`WidgetsApp` `builder:` wraps the Navigator, so a
  `Stack` there (app, then the toggle) sits above every route and dialog.
- **Interception:** in the same `Stack`, while on, a full-screen `Listener` with
  `HitTestBehavior.opaque` (inside an `AbsorbPointer` for the app) takes every pointer event.
- **Hit test:** hit-test the app's render tree at the position (`WidgetsBinding.instance` hit
  testing in the current view); map the deepest render object to its creating element/widget via
  its debug creator.
- **Exact source:** the widget's creation location (see the table above).
- **Signature:** `id` = the widget's `Key` value (`ValueKey('x')`, `Key('x')`); `names` = custom
  widget type names on the path (skip framework types like `Padding`, `Container`); `text` from
  `Text`/`RichText`; `attrs` = `Semantics` label.
- **Screen:** the current route's `settings.name`, else the top route's page widget type.
- **Birth:** `Key('x')`, `ValueKey('x')`, `class X extends StatelessWidget/StatefulWidget`, the
  constructor call `X(`.
- **Clipboard:** `Clipboard.setData(ClipboardData(text: line))`.
- **Reaching the locator:** as React Native (emulator via `adb reverse`, simulator `localhost`,
  phone over LAN with the token).
- **UI-test driver:** `integration_test`, `flutter_driver`, Patrol.

## Android — Views (XML layouts)

- **Debug switch:** `BuildConfig.DEBUG`; put the tapper in the `debug` source set so a release
  build does not even contain it.
- **Toggle layer:** dialogs and popups are separate windows above the activity, so a button in the
  activity cannot reach over them. The toggle that always works: a **debug notification with an
  on/off action**. Add a floating button in the activity's decor view for convenience.
- **Interception:** wrap each window's `Window.Callback` (`dispatchTouchEvent`, `dispatchKeyEvent`)
  — activity windows via `Application.registerActivityLifecycleCallbacks`; consume every event
  while on.
- **Hit test:** descend from the window's decor view to the deepest visible child whose bounds
  contain the point.
- **Signature:** `id` = `resources.getResourceEntryName(view.id)` when the view has an id —
  **also add the camelCase form** (`buy_now` → `buyNow`), which is what view binding writes;
  `names` = the view's class simple name when it is the app's own; `text` (TextView);
  `attrs` = `contentDescription`, `tag` when a string.
- **Screen:** the activity's class, plus the visible fragment's class; their layout files
  (`setContentView(R.layout.x)`, `inflate(R.layout.x`) count as screen files too.
- **Birth:** `android:id="@+id/x"` in layouts; for views built in code, the constructor
  `X(context` and `id = R.id.x`. `findViewById(R.id.x)` and `binding.x` are references.
- **Text table:** `res/values*/strings.xml` → `@string/key`, `R.string.key`,
  `getString(R.string.key)`.
- **Clipboard:** `ClipboardManager.setPrimaryClip(ClipData.newPlainText("tap2line", line))`.
- **Reaching the locator:** `adb reverse tcp:<port> tcp:<port>`, then `http://127.0.0.1:<port>`.
  Cleartext HTTP is blocked by default: allow it for `127.0.0.1` only, in a **debug-only**
  network security config. Phone over Wi-Fi: LAN address plus token.
- **UI-test driver:** Espresso, UI Automator, Maestro.

## Android — Jetpack Compose

- As Views for the debug switch, the toggle, interception, the clipboard and the locator.
- **Hit test and signature:** a `ComposeView` is one view; read its **semantics tree** (the one
  UI tests use, reachable from the view's test root) and take the deepest node whose bounds contain
  the point. `id` = `testTag`; `text`; `attrs` = `contentDescription`, role. Composable function
  names are not in the semantics tree — rank on what is there.
- **Birth:** `Modifier.testTag("x")`, `@Composable fun X(`, the call `X(`.
- **Exact source:** see the table above. Without it, elements with no test tag and no literal text
  come back "not found" — say so, and suggest test tags, which also help UI tests.

## iOS — UIKit

- **Debug switch:** `#if DEBUG`.
- **Toggle layer:** a separate `UIWindow` with a window level above alerts, holding the toggle —
  it stays above every presented controller and alert.
- **Interception:** while on, that overlay window covers the screen and takes every touch
  (`hitTest` returns itself); then hit-test the app's key window at the same point
  (`window.hitTest(point, with: nil)`) to find the element.
- **Signature:** `id` = `accessibilityIdentifier`; `names` = the view's class name when it is the
  app's own; `text` = title/text/label; `attrs` = `accessibilityLabel`.
- **Screen:** the class of the topmost presented/visible view controller.
- **Birth:** `accessibilityIdentifier = "x"`; in storyboards/xibs the element's `<accessibility …
  identifier="x"/>` and its `userLabel`; `class X: UIView`, the call `X(`.
- **Text table:** `Localizable.strings` / `.xcstrings` → `NSLocalizedString("key"`,
  `String(localized: "key"`.
- **Clipboard:** `UIPasteboard.general.string = line`.
- **Reaching the locator:** the simulator shares the Mac's network — `localhost`. A phone: the
  Mac's LAN address plus the token; App Transport Security needs local networking allowed in the
  **debug** configuration only.
- **UI-test driver:** XCUITest, Maestro.

## iOS — SwiftUI

- As UIKit for the switch, the overlay window, the clipboard and the locator.
- **Hit test and signature:** SwiftUI views are not `UIView`s; a UIKit hit test lands in hosting
  internals. Use the **accessibility tree** of the hosting view instead: the deepest accessibility
  element whose frame contains the point. `id` = `accessibilityIdentifier`; `text` = label.
- **Birth:** `.accessibilityIdentifier("x")`, `struct X: View`, the call `X(`.
- No exact source exists; elements without an identifier or literal text come back "not found".
  Say so, and suggest identifiers, which also help XCUITest.

## Desktop — Qt, .NET

- **Qt Widgets:** debug via `#ifndef QT_NO_DEBUG`/a debug build option; an application-wide event
  filter takes mouse events while on; `QApplication::widgetAt(globalPos)`; `id` = `objectName`;
  `names` = `metaObject()->className()` when it is the app's own; `text`; birth
  `setObjectName("x")`, `.ui` files' `<widget class="…" name="x">`; screen = the top-level
  window's class; clipboard `QGuiApplication::clipboard()->setText`.
- **QML:** `objectName: "x"` and the QML `id` (reachable through the item's context) as `id`;
  birth `id: x`, `objectName: "x"`; the file of the component (`X.qml`) from the item's type name.
- **WPF:** a class handler for `PreviewMouseDown` on `Window` (`EventManager.RegisterClassHandler`)
  takes clicks while on; `VisualTreeHelper.HitTest`; `id` = `x:Name` (`FrameworkElement.Name`)
  or `AutomationProperties.AutomationId`; birth `x:Name="x"`, `AutomationId="x"`; clipboard
  `Clipboard.SetText`; debug `#if DEBUG`.
- **WinForms:** `Application.AddMessageFilter` while on; `Control.Name`; birth in the designer
  file (`this.x.Name = "x"`).
- **Reaching the locator:** `localhost`.
- **UI-test driver:** Qt Test, FlaUI/WinAppDriver.

## Any other UI

Answer the same eleven questions from the toolkit's documentation, write them into the hand-over
doc in the same shape as a section above, and build from that. The tool needs only three things
from a toolkit: a way to see a tap before the app does, a tree of elements under a point, and
something on each element that also appears in the source.
