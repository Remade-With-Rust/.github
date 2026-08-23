---
name: building-the-new-internet
description: >-
  The single house playbook for starting and shipping a Rust app, service, or website that is
  ready to deploy onto the MATA distributed cloud (disco) the day it goes live. Answers "what
  do we use for X" with the fixed stack — mID for identity/access, SpaceDB for storage,
  remade_ffmpeg_rs for media, FFAI for AI, rusty_alloc for allocation, rusty_zstd for
  compression, thoth/rusty_tokens/rusty_symbols/rusty_a11y for UI chrome, Deputy for the
  supply chain — plus the architecture (API-first, general primitives, per-entry CRDT, trust
  boundary), the deploy-readiness seams, the performance doctrine, and the build/validate
  workflow. Read BEFORE `cargo new`, before adding any dependency, before choosing a
  crypto/storage/identity/AI/allocator stack, and before a PR that adds one. Consolidates the
  former rusty-coding-requirements, rusty-dev, rusty-blazing-fast, rusty-memory-model,
  rusty-unsafe-optimizations and rusty-async-internals skills into one.
---

# Building the New Internet

You are starting, or growing, a Rust application, service, or website. This skill is the whole
contract: the stack you pick from, the shape you build in, and what "ready to deploy" means.

The target is not a VPS. It is the **MATA distributed cloud** — a mesh of machines nobody owns
centrally, where your data lives near your users, every access is a signed capability, and the
node running your code is hardware you do not control. Build for that from day one and the
deploy is a command. Retrofit it later and it is a rewrite.

---

## 0. The one-line test

> **Could this ship as-is to a user who assumes their data is theirs alone, onto a machine you
> do not own, with no C toolchain anywhere in the build?**

Every rule below is a corollary. When a decision is unclear, ask that sentence.

Three failures it catches, in the order they usually happen:

| You wrote | It fails because | Fix |
|---|---|---|
| A Postgres connection string | there is no data centre to connect to | SpaceDB replica (`stack.md` §2) |
| `openssl-sys` / `protoc` / `libgmp` in the tree | a mesh node has no C toolchain, and `wasm32` has none at all | the replacement table (`stack.md` §8) |
| A session cookie backed by a users table | the user's identity is a keypair they hold, not a row you own | mID (`stack.md` §1) |

---

## 1. The stack — what we use for X

Reach for the house crate **first**. Each exists to delete a C dependency, a data centre, or a
password. Full detail, versions and traps: **`stack.md`**.

| Need | Use | Status today |
|---|---|---|
| **Identity, sign-in, access control** | **mID** — `mid-verify` (RP side), `mid-issuer` (wallet), `mata-cap` (authz) | published, MIT/Apache |
| **Storage — all of it** | **SpaceDB** — `spacedb-sdk` | `0.5.2` on crates.io |
| **Media convert / transcode / probe** | **remade_ffmpeg_rs** — the `rff` facade crate, **by git** | pre-1.0, git only — see the trap below |
| **AI — OCR, ASR/TTS, detection, VLM** | **FFAI** — `ffai-core` + the engine crate you need | published |
| **AI — LLM serving / tool calling** | **mistral.rs** on **candle**, behind one `ChatEngine` seam | — |
| **Memory allocation** | **rusty_alloc**, via a one-crate seam (or `rusty_alloc_default`) | `1.0.1` on crates.io |
| **Compression** | **rusty_zstd** | deploying — **not on crates.io yet**, git/path only |
| **UI chrome — glyphs, tokens, a11y** | **thoth** (or split: `rusty_symbols` / `rusty_tokens` / `rusty_a11y`) | published |
| **UI framework** | **Dioxus** — web, PWA, desktop, mobile from one codebase | — |
| **Supply chain** | **Deputy** — `cargo install deputy-cli` | published |
| **Crypto** | **RustCrypto** — Argon2id + AES-256-GCM | — |
| **TLS, system tooling** | **memorysafety.org** — rustls, sudo-rs, curl-rust | — |
| **Text shaping / layout / raster** | **OxiText** (never FreeType/HarfBuzz) | — |
| **Internal wire format** | **oxicode** (never on public APIs — those get JSON) | — |
| **Big / exact math** | **OxiNum** (never on secrets — not constant-time) | — |
| **Protobuf codegen** | **OxiProto** (no system `protoc`) | — |

> **Two traps that cost real time.**
> **`rff` on crates.io is not ours** — it is an unrelated fuzzy text selector. Depend on
> remade_ffmpeg_rs **by git URL**, never by bare crate name.
> **`rusty_zstd` is not published yet.** Use a git or path dependency and pin a commit; when it
> lands on crates.io, switch the pin, don't switch the API.

**Before anything not on this table**, walk the ladder: memorysafety.org → RustCrypto →
[Remade-With-Rust](https://github.com/orgs/Remade-With-Rust/repositories) → cool-japan `Oxi*` →
crates.io. The org ships new crates continuously — **check it before you write the thing
yourself** (`stack.md` §8 has the current inventory and how to re-check it).

Adding a `*-sys` crate is a decision you state out loud in the PR, with the reason it could not
be avoided.

---

## 2. Day one — the scaffold that is already deploy-ready

```
myapp/
├── Cargo.toml            # workspace; pins live here, once
├── crates/
│   ├── myapp-core/       # LIBRARY: the ops. No allocator, no UI, no I/O policy.
│   ├── myapp-api/        # LIBRARY: typed ops over a Transport seam. API first.
│   ├── myapp-cli/        # DELIVERABLE: allocator declared here
│   ├── myapp-server/     # DELIVERABLE: allocator declared here
│   ├── myapp-ui/         # DELIVERABLE (Dioxus): allocator declared here
│   └── myapp-alloc/      # the allocator seam — one crate, one pin
```

Four rules the layout encodes, and why each is load-bearing:

1. **`#[global_allocator]` lives in the deliverable, never a library.** A program may define
   exactly one. A library that declares it forces the choice on every consumer and makes two
   such libraries impossible to link together. In a diff, `#[global_allocator]` in a library
   crate is a stop-the-review finding. (`stack.md` §5)
2. **The core knows bytes, CIDs, DIDs and capabilities — never a product type.** Litmus: *could
   a developer who has never heard of your product use this op?* (`architecture.md` §1)
3. **Every capability is an op before it is a button.** The UI is consumer #1, never the only
   consumer. If a behaviour only exists inside an event handler, it is not built yet.
   (`architecture.md` §2)
4. **Every persisted type goes in per-entry, encrypted, under its own compound key.** No
   single-blob formats, ever. (`architecture.md` §3 — this one is a law, not a preference)

Copy-paste scaffold, manifests and the allocator seam: **`stack.md` §9**.

---

## 3. The five seams that make deploy a command

The distributed cloud does not ask you to port your app. It asks you to have built against five
seams it fills. Build against them locally on day one and `disco` takes over on launch day.

| Seam | You build against | disco fills it with |
|---|---|---|
| **Identity** | `mid-verify` — verify a token, locally, no network | the mID roster + KMS resolver |
| **Storage** | `spacedb-sdk` — a local replica on your own box | `managed-db`, mesh-replicated |
| **Transport** | SpaceDB's `Transport` trait | iroh + relay across the mesh |
| **Placement** | SpaceDB's `ShardStore` | erasure shards, anti-affinity, self-repair |
| **Settlement** | SpaceDB's `Settlement` trait | Iron Bank, `$MATA` per-use metering |

Then the go-live is:

```sh
disco sites deploy ./dist --domain app.example.com
# seals -> chunks -> places -> replicates -> registers -> serves off the mesh
```

**Be honest about what is live.** `identity` is GA. `hosting`, `edge-functions`,
`object-storage`, `cdn`, `bandwidth`, `dns` and `managed-db` are **Preview** — the primitives
are built and the provide→prove→PAID loop closes, but the self-serve console path is still
being wired. Build against the seams now; the switch flips under you.
Full deploy path, op by op, plus what to run today: **`deploy.md`**.

---

## 4. Reference files — read the one your task touches

- **`stack.md`** — every dependency decision with versions, features and traps: mID, SpaceDB,
  media, AI (the candle/mistral.rs two-layer rule), the allocator seam, compression, the UI
  crates, Deputy, the Oxi* replacements, and how to check the org for what shipped since.
- **`architecture.md`** — general primitives / thin wiring, the op→SDK→gateway seam, API-first,
  the per-entry CRDT law, the trust boundary, content-addressing, and *cores are sound, bugs
  live at the seams*.
- **`deploy.md`** — the road to the distributed cloud: mID sign-in end to end, SpaceDB from
  local replica to mesh, `disco sites deploy`, the resource SDK's six services, what is
  Available vs Preview, and the deploy-readiness checklist.
- **`performance.md`** — making it fast without breaking it: the five high-leverage moves, the
  measurement bar a number must clear before you act on it, when `unsafe` is earned and how to
  fence it, and the memory model that explains why the rules are shaped this way.
- **`workflow.md`** — the build/validate/debug loop: compile-gate every target, validate the
  mechanism in isolation, why a green test can test the wrong scenario, instrument before you
  guess, and the curiosity discipline for when a result defies expectation.
- **`ui.md`** — Dioxus 0.7 footguns across web/desktop/wasm, the thoth glyph/token/a11y trio,
  and the staged-progress pattern that turns "it silently does nothing" into a visible failure.

---

## 5. Pre-flight — run this before you add a dependency or open a PR

- [ ] Does it build for **every** target you ship, including `wasm32-unknown-unknown`, with no
      system C library, no `protoc`, no CMake?
- [ ] Did I check the org (§1 ladder) before reaching for crates.io — including for something
      that shipped since I last looked?
- [ ] Does every new persisted type go in **per-entry**, encrypted, under its own compound key?
- [ ] Is the capability reachable from an **op** — callable by a CLI, a test and an agent — not
      only from the UI?
- [ ] Does any **library** crate in this diff declare `#[global_allocator]` or depend on
      `rusty_alloc-api` directly instead of through the seam? **Stop the review.**
- [ ] ML change: right layer — mistral.rs for LLM serving, candle for everything else — and do
      `candle-core` / `-nn` / `-transformers` still move in lockstep?
- [ ] Secrets: Argon2id-derived, AES-256-GCM at rest, never touched by non-constant-time math
      (OxiNum), never sent to a remote model?
- [ ] Tests written **and run**, against the real deployment topology — not a green test of the
      wrong scenario?
- [ ] `cargo check` on every target the change touches, before the push?
- [ ] No `unwrap()` on a path a user can reach; no "temporary" shim; no TODO left as the fix?
