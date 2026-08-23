# ui.md — the surface: Dioxus, and the chrome crates that come before it

One codebase across web, PWA, native desktop (Windows/macOS) and mobile (iOS/Android).
"Cross-platform" means all of them — "works on my target" is not working.

Remember the ordering rule from `architecture.md` §2: **the op exists before the button.** This
file is about the button.

---

## 1. Chrome first — don't hand-roll glyphs, colours or ARIA

Before your first component, take the three chrome crates. They are `no_std`, wasm-checked, and
each defaults to installing `rusty_alloc` (opt out with `default-features = false` — see
`stack.md` §5).

```toml
# unified
thoth = { git = "https://github.com/Remade-With-Rust/thoth.git", tag = "v0.3.0" }
# or split
rusty_symbols = "0.1"
rusty_tokens  = "0.2"
rusty_a11y    = "0.2"
```

**`rusty_symbols` — glyphs as named constants.** Application `.rs` files stay ASCII (`\u{...}`
escapes), so a Windows-1252 round-trip cannot mojibake your icons, and VS15 pinning makes a
WebView render the same glyph on every platform. One source of truth instead of a raw literal
scattered at every site.

**`rusty_tokens` — the theme contract.** Token names *are* the CSS API (`--rt-color-fg`).
Defaults are a small neutral starter; apps override in CSS. `css::root_sheet()` emits the
`:root` block to inject into a WebView or Dioxus app.

**`rusty_a11y` — chrome a screen reader can hear.** Labelled glyphs get `role="img"` names;
sync / saved / offline updates ride polite or assertive live regions. Small HTML string
builders — no DOM crate, no JS.

---

## 2. Dioxus 0.7 footguns — the ones that cost hours

### Dynamic `style:` does NOT re-apply on re-render. Dynamic `class:` does.

The single most expensive UI bug here. A dynamic **`style:`** string attribute is applied **on
mount only** — when a signal changes and the component re-renders, the new string is *not*
re-applied to the existing DOM node.

```rust
// Reverts / doesn't update on re-render:
div { style: if active() { "color: red" } else { "color: gray" } }

// Updates correctly:
div { class: if active() { "text-accent border-b border-accent" }
              else        { "text-fg/50 border-b border-transparent" } }
```

Symptom: a tab or underline "reverts to the old look" on click even though the signal changed.
Drive visual state with a dynamic `class`. If you genuinely need dynamic inline style, change
the element's `key` to force a remount.

### A prebuilt `tailwind.css` makes new utility classes silently no-op

A desktop build ships a **prebuilt** `assets/tailwind.css` containing only the classes present
at build time. A class you add in RSX that is not already in that file — `border-b-2`, `-mb-px`,
`gap-0.5` — **silently does nothing**. No error, no style. `py-2.5` may be present while
`gap-0.5` is not.

Before blaming your logic, confirm the class exists. Grep with `-F` because of the escaped dot:
`grep -F "py-2\.5" tailwind.css`. Fix by regenerating Tailwind, or use a class already compiled
in.

### What `dx serve` actually serves — this decides where files go

| you have | result |
|---|---|
| `asset!("/assets/x.css")` | hashed path (`/assets/x-dxh<hash>.css`), **served** — the only way `assets/` files get served in dev |
| a raw file in `assets/`, not referenced via `asset!()` | **404 in dev serve** — even the template's own fonts and images; they only appear in the built SSG output |
| anything in `public/` | served **verbatim at the web root**: `public/uploads/x.jpg` → `/uploads/x.jpg` |

**Put runtime, uploaded and static files in `public/`**, so the same clean path works in the
live preview *and* the deployed bundle. Have your build script copy `public/*` into the output
too. Verify serving empirically with marker files and `curl` before you design storage paths.

### `[web.resource.dev]` is dev-only injection

In `Dioxus.toml`, `[web.resource.dev] script = ["…"]` injects `<script src>` tags **only during
`dx serve`** — never in `dx bundle --ssg` or a release build. Ideal for dev-only tooling.
Absolute URLs work, so an app-run localhost server can serve the script and bake per-session
config into it, sidestepping the dev asset pipeline entirely.

### Scripts injected in `<head>` run before `<body>` exists

They execute during head parse, so `document.body` is **null** and `document.body.appendChild()`
throws. Your script dies *after loading but before drawing anything* — a 200 with no visible
effect, easy to misdiagnose. Always defer DOM work:

```js
function mount(){ /* touch document.body here */ }
if (document.readyState === "loading") document.addEventListener("DOMContentLoaded", mount);
else mount();
```

Dioxus mounts the app into `#main`, not `<body>`, so elements you append to `document.body`
survive re-renders — but they must be added after the body exists.

### RSX interpolation traps

- Nested escaped quotes or an `if`/`else` inside a `"{ … }"` interpolation is fragile to parse.
  **Precompute display values before the `rsx!`** and interpolate a plain binding:
  ```rust
  let label = if p.route == "/" { "Home".to_string() } else { p.route.clone() };
  rsx! { div { "{label}" } }
  ```
- `for x in items` in RSX takes rsx-node bodies, not arbitrary statements. Precompute a `Vec` of
  display tuples and iterate that.

### Big inline JS/HTML blobs: a const plus `.replace()`, never `format!`

`format!` requires doubling every `{` and `}` in a JS blob — error-prone hell.

```rust
const JS: &str = r####"(function(){ var CFG={endpoint:"__EP__",token:"__TOK__"}; … })();"####;
JS.replace("__EP__", ep).replace("__TOK__", tok)
```

Syntax-check the result with `node --check` before shipping it.

### Desktop is a system webview (`dioxus://` origin)

- An `http://localhost:<port>` **iframe** inside the webview can be blocked by mixed-origin/CSP
  rules — the pane comes up blank while the dev server is perfectly fine. Prefer opening the
  preview in the real browser over fighting the webview.
- **OAuth and third-party flows cannot run inside the `dioxus://` origin.** Open them in the
  system browser. (With mID you mostly avoid this class of problem entirely — see `stack.md`
  §1.)

---

## 3. The staged-progress pattern — UX *and* debuggability in one

A button that does `spawn(async { multi_stage_thing().await })` and changes nothing on screen is
bad UX **and un-debuggable** — you cannot tell a stall from a no-op from a wrong-mode path. This
cost hours once: a Play button silently sent nothing, invisibly, because it was in mock mode.

Give the async task a `Signal<Progress>` it updates at **each real stage**:

```rust
#[derive(Clone, PartialEq)]
struct Progress { active: bool, phase: Phase, elapsed_ms: u128, done: Vec<(String, u128)> }

async fn run(mut progress: Signal<Progress>) {
    let start = std::time::Instant::now();          // Instant is Copy -> moves into the task
    let mut set = move |phase, done| progress.set(Progress {
        active: true, phase, elapsed_ms: start.elapsed().as_millis(), done });

    for attempt in 1..=6 { set(Phase::Resuming { attempt }, done.clone()); /* retry */ }
    done.push(("stage 1 done".into(), start.elapsed().as_millis()));
    // ...stage 2 (poll), stage 3 (launch)
}
// onclick: progress.set(Progress::starting()); spawn(run(progress));
// render:  if progress().active { ProgressView { progress } } else { button { "Play" } }
```

- **Update on each loop iteration** (retry attempt, poll tick) so elapsed time and stage detail
  move without a separate UI ticker.
- **Make the failure branch informative.** "no target paired" tells you instantly it is mock
  mode. **A progress view *is* a diagnostic** — surface the boundary state, don't hide it.
- `Signal` is `Copy`. Capture it into the spawned task by value and `.set()` from the task; that
  triggers a re-render. A `move` closure *copies* a `Copy` `Instant`/`Signal`, so both stay
  usable outside it.

Pair this with a greppable marker at each hop on the daemon side (`workflow.md` §4).

---

## 4. Patterns worth copying

**Multi-phase flow in one component — early-return by state, most terminal first:**

```rust
if let Some(oc) = outcome() { return rsx!{ /* terminal   */ }; }
if let Some(cx) = pending() { return rsx!{ /* passphrase */ }; }
if let Some(s)  = session() { return rsx!{ /* live view  */ }; }
rsx!{ /* start screen */ }
```

Clone the fields you need out of the signal at the top of each branch. Capture only owned or
`Copy` values into `onclick` closures — a `Vec`/`String` read from a signal must be `.clone()`d
per closure; `Signal` itself is `Copy`.

**Live counters from a broadcast channel:** spawn **one** task when the flow starts; loop
`rx.recv().await`; **re-check the flow is still active via a signal before processing** so a
cancel stops it; dedupe and push into a `Signal<Vec<_>>` the view reads `.len()` of. Treat a
`Lagged` error as "keep going", not as fatal. Caveat: `recv().await` blocks until the next
event, so a cancelled task lingers until one arrives. Fine for low-rate control events; use
select-plus-cancel if it matters.

**Closing a dialog from an async task:** calling an `EventHandler` prop from inside `spawn` is
awkward. If the parent just flips a `GlobalSignal`, write it directly from the task —
`*SHOW_X_DIALOG.write() = false;`. Works from anywhere, no handler threading.

**`spawn` placement:** it works at an event handler's top level. Don't rely on a nested `spawn`
inside an already-spawned task having the reactive scope — set up channels and consumers at the
handler top level instead.

---

## 5. The UI is a client, not the boundary

Restating the rule from `architecture.md` §6, because this is where it gets violated: **the
process holding the vault key, the database or the provider keys is the trust boundary.** A UI —
web, desktop shell, browser bubble, mobile app — relays *intent* over localhost and never holds
keys.

Gate it three ways: per-session token baked into the served client, localhost bind, explicit
CORS. `Access-Control-Allow-Origin: *` means any page can POST, so the token is the real defence
— it must reach the legitimate client and nothing else.
