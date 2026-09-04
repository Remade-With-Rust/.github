# workflow.md — the build, validate and debug loop

The habits below exist because each one caught a real break that a normal-looking green build
did not. They are cheap. Run them.

---

## 1. Compile-gate every target, before every push

```sh
cargo check -p <crate> --target <your-native-triple>
cargo check -p <crate> --target wasm32-unknown-unknown     # anything the UI touches
```

This catches the classic break: a type gains a field while a call site still constructs it the
old way — which can already be committed by someone else. **A two-minute gate beats a broken
main.**

### The wasm-default-target trap

If the workspace `.cargo/config.toml` sets `build.target = "wasm32-unknown-unknown"`, then a
bare `cargo check` / `cargo test` builds for **wasm**, and native-only crates (tokio net, mio,
axum, anything with a server) fail with cryptic errors — *"This wasm target is unsupported by
mio"*, `unresolved import crate::sys::tcp`.

> When a test run explodes with mio/wasm errors, it is almost always a missing `--target`, not a
> real bug.

Always pass the native target explicitly for native crates and tests.

### Target-gated deps → mirror structs in the UI

A native-only crate is declared under `[target.'cfg(not(target_arch = "wasm32"))'.dependencies]`.
The UI compiles for **both** targets, so:

- UI signal types must be **local mirror structs**, never the native crate's types.
- Every call into the native crate is `#[cfg(not(target_arch = "wasm32"))]`.
- Provide a **wasm stub** for any `pub fn` the UI calls on both targets:
  ```rust
  #[cfg(target_arch = "wasm32")]
  pub async fn crawl_site(_: &str) -> Result<Report, String> { Err("desktop only".into()) }
  ```
- **Heavy crypto is a target-gate too — for bundle size, not just buildability.** Keep BLS /
  pairing crates out of the wasm thin-client bundle by gating both the dependency *and* the
  module.

### Cargo trivia that costs an hour each

- **Features merge across every declaration of a dep.** A base `tokio = { features = ["sync"] }`
  plus a native `tokio = { features = ["time"] }` gives native code `sync + time`. Check *all*
  declarations — there may be three or more.
- **`!Send` types cannot cross `.await` or a thread boundary.** Keep them inside sync helper fns
  that take `&str` and return owned data, called *between* awaits. To let a background thread
  trigger privileged work in a UI-thread-bound context, bridge with a channel carrying only
  `Send` data: text crosses, auth stays put.
- **`#[tokio::test]` defaults to a current-thread runtime.** Anything needing real concurrency
  needs `#[tokio::test(flavor = "multi_thread")]`.
- **Adding a workspace crate means registering it in more than one place.** Grep for an existing
  crate's name to find all of them.
- **Stale cargo locks hang everything.** A killed build can leave the global package-cache lock
  held, so any `cargo metadata` or build blocks indefinitely. Symptom: no output, no CPU. Kill
  the orphaned process.

---

## 2. Validate the mechanism in isolation, before the app

The app needs auth, a full build and a browser. The *mechanism* usually needs none of that.

- **A standalone harness crate in a scratch dir** (not the repo) — replicate the parser, the
  crawler, the localhost server, and `curl` it. Seconds, not minutes.
- **A pure end-to-end data-flow test with no network.** Dev-depend the *same* primitive the app
  uses, use a serde round-trip as "the publish", hold bytes as "a host". Assert the **strong**
  property — the recovered key decrypts *real data*, not merely that key bytes match — plus the
  adversarial cases: below-threshold quorum, tampered blob → AEAD reject, wrong passphrase →
  fail closed.
- **`curl` the real routes** for status codes and wiring, minus the parts only the app has.
- **`node --check`** any generated or injected JS before shipping it.

Prove the moving part works, *then* wire it into the app. When something fails after that, you
know which half it is in.

---

## 3. A green test can test the wrong scenario

> **The question to ask of every passing test: does this exercise the actual deployment
> topology?**

A test that shares state **in-process** — one `Arc`, one connection — can hide the real
cross-process gap completely. A shared-registry test was green with an in-process
`Arc<InMemory…>` while two real gateway processes could not share an embedded store *at all*.
An embedded database engine typically takes a process-exclusive file lock; two processes on the
same path simply cannot both open it.

Before believing a gap is closed, add a test that opens **two independent connections** to the
real backing, or is honestly env-gated against a live server. See `architecture.md` §9.

---

## 4. When something "silently does nothing", instrument before you guess

Hours went to a Play button that sent nothing — invisible because there was no progress UI and
nobody read the daemon log.

- **On the client:** a staged progress `Signal` for every long async action. It is a UX feature
  and a debugging feature at the same time, and it turns "it's stuck somewhere" into "it's stuck
  *here*".
- **On the server/daemon:** a greppable marker at each hop, so you can find the last one that
  fired.
- **In a long remote chain, each fix reveals the next blocker.** Instrument the boundary so the
  *next* one is visible, rather than re-guessing from the top each time.

---

## 5. The curiosity discipline — a refuted hypothesis is where the finding lives

> **When a result defies the expected outcome, you have not hit a snag. You have been handed a
> pointer. Spend it where you are standing** — that is the one place the context is already
> loaded.

Two bounded moves do almost all the work.

### Move 1 — descend exactly ONE layer

The stage is not the function; the function is not the loop; the loop is not the instruction.
When a number surprises you, the next question is not "why is this slow" but "what is this
*actually made of*, and which piece is anomalous?" **One layer, not five** — descending too fast
is how you end up optimizing a mechanism you never confirmed was running.

**The tell that you owe a descent: the arithmetic does not close.** If a ratio implies a
per-unit cost that could not plausibly be true of the instructions involved, your
*decomposition* is wrong — not the code. A measured 0.259 probes/byte against a reference's ~1.0
while running 2× slower implies a 7.7× per-probe cost for five identical instructions. That is
not a finding; it is a receipt saying *"the thing you are calling a probe is not what is eating
the time."*

**Reach for a single-variable instrument** — an input where all but one term collapses to zero.
Do not solve simultaneous equations across a corpus when one file makes the system trivial.

### Move 2 — read the SIBLINGS (this is the one people skip)

> **When you open a file to test hypothesis X and X turns out to be FALSE, do not close the
> file.**

A refutation feels like a dead end, so the instinct is to back out and look elsewhere. But
consider what you hold at that moment: you navigated to the exact site that matters, you loaded
the surrounding context, you now know one specific thing that is *not* the cause — and reading
the ten lines around your cursor costs approximately nothing.

**The refutation already paid for the trip.** Leaving without looking around throws away the
only expensive part of the expedition.

| you came to check | also read |
|---|---|
| one field of a struct | every other field allocated beside it |
| one function | the functions immediately above and below |
| one allocation | what else the same constructor allocates |
| one call site | the other call sites of the same function |
| one counter | the counters incremented next to it |

**Check units before mechanism.** Most impossible values are a unit error, not a discovery.

**Bound the wander:** one layer, the siblings, then report.

---

## 6. Reviewing and auditing — verify every headline claim first-hand

Fanning out across a codebase is great at *locating* evidence fast. It is unreliable at grading
it: **broad sweeps systematically over-rate severity.** Findings that dissolve on a first-hand
read are the norm, not the exception.

- **Fan out by dimension** — one pass per subsystem or theme, each returning `file:line`
  evidence, a severity, and a "done right" section.
- **Then read the decision points yourself** for every High and Critical, before writing it up.
  Real reversals from doing this: "the endpoint id is self-claimed" (wrong — it is
  QUIC-authenticated); "double-settle race" (not exploitable under the `&mut self` mutex);
  "callback runs arbitrary code" (bounded by audience binding → Medium, not Critical). The
  first-hand pass *sharpens* the real ones too.
- **Separate a bug from a deferred-by-design gate.** "Don't ship multi-user until updates are
  signed" is a gate to respect, not a fix to make. The vulnerability is shipping the feature
  before the gate — label it that way.
- **Collapse N findings into the few actual fixes.** 79 findings was ≈ 10 fix-families; most
  Mediums are "surface X hasn't adopted the pattern surface Y already proves".
- **State the scope you did not cover.** "No findings here" is not "clean" if you never looked.

---

## 7. Tooling baseline

Verify what is already configured before adding any of it.

**Local loop.** `bacon` (watch-runs cargo on save) and `cargo nextest` (parallel, per-test
isolation, cleaner output). Complements — does not replace — the dual-target `cargo check` gate.

**Debug ergonomics.** `dbg!(x)` over `println!` (prints the expression, `file:line` and a pretty
value, and returns the value so it is drop-in). `todo!()` / `unimplemented!()` over a `// TODO`
comment — they compile now and panic loudly if hit. Strip `dbg!` before committing.

**Lints as gates.** `cargo clippy` in CI, escalated to deny for the ones that matter
(`clippy::unwrap_used`, needless clone, inefficient loops). `#[must_use]` on any type whose
return must not be dropped — a built `Builder`, a capability, a grant — so ignoring it is a
compile error.

**Reproducibility.** `rustfmt.toml` at the root; `rust-toolchain.toml` pinning the exact compiler
and components, so every machine and CI runner builds identically.

**Supply chain.** `cargo audit` (RustSec CVE scan), `cargo deny` (ban list, license allowlist,
duplicate-version detection), and **Deputy** for the owned-vault layer (`stack.md` §8). On a mesh
codebase the dependency surface *is* the attack surface.

**Two common tips deliberately NOT adopted** — recorded so they are not re-litigated:

- *"Start with trait objects, switch to generics for performance"* is backwards as a default.
  Prefer generics and static dispatch (zero-cost, monomorphized); drop to `dyn` deliberately, for
  heterogeneous collections or to cut compile time and binary size.
- *"Move repeated methods into a macro"* is over-eager. A `macro_rules!` that generates
  `find`/`save`/`delete` breaks go-to-definition and readability. Prefer a trait with default
  methods or a generic; reach for a declarative macro only when a trait genuinely cannot express
  the pattern.

---

## 8. Keep the repo clean

Scratch harnesses, probe binaries and one-off logs go in a scratch directory **outside the
repo**. If a probe is worth keeping, it becomes an example or a test with a name that says what
it answers — not an untracked file at the root that nobody dares delete a year later.
