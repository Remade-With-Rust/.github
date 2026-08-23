# stack.md — every dependency decision, with the traps

The table in `SKILL.md` §1 is the summary. This is the detail: exact crate names, versions,
features, the reason each choice exists, and the mistake each one prevents.

---

## 1. Identity and access — mID

**Repo:** https://github.com/Remade-With-Rust/mid · MIT OR Apache-2.0 · native + `wasm32`

A user's identity is **a keypair they hold on their own device** — not a row in your database,
not an account on a portal. They hand your service a token signed by their own key; you verify
it **entirely locally**, with zero runtime calls to MATA or anyone.

| Crate | Use it for |
|---|---|
| `mid-verify` | **Start here.** RP-side verifier: genesis self-sig → roster chain → head version → JWS. A pure function, no I/O. |
| `mid-issuer` | Wallet side — mints the self-issued sign-in JWT from an identity snapshot + device key. |
| `mata-sign` | Sign arbitrary content as a `did:mata`, verifiable offline by anyone. |
| `mata-identity` | The user-owned keypair — the `did:mata` root. |
| `mata-cap` | One `Capability` / `Caller` / `authorize` model, so every service gates the same way. |
| `kms-client` / `kms-verifier` / `kms-types` | Sovereign-auth envelopes; `kms-client` is `wasm32` + native. |

Browser/Node side is `@matanetwork/sovereign-id` — same protocol, same wire format. Issue on
one, verify on the other.

```toml
[dependencies]
mid-verify = "0.1"
```

```rust
use mid_verify::{verify_mid_response, VerifyConfig};

let config = VerifyConfig {
    expected_audience: "https://acme.com".into(), // your origin; must equal the token's `aud`
    expected_nonce:    session_nonce,             // the single-use nonce you issued
    max_iat_skew_secs: 120,
    now_unix_secs:     now,                        // you supply the clock — verify stays pure
};

let verified = verify_mid_response(jwt, &config)?;
// verified.did                 -> stable user id (your users-table primary key)
// verified.claims              -> only what the user consented to disclose
// verified.genesis_roster_hash -> anchor; one DID always presents the same hash

// Defeat stolen-device replay. Do not skip this line.
verified.check_rollback(last_seen_version)?;
```

**What you do NOT build:** `client_id`, redirect-URI allowlists, a `/token` back-channel, a
JWKS endpoint, a DID-resolver HTTP call, MAU metering, password reset, or a session table. The
token carries its own resolution data and the DID *is* the public key.

**Traps.**
- Skipping `check_rollback` leaves stolen-device replay open. It is one line.
- `expected_audience` must be your real origin. A wildcard here is an auth bypass.
- The nonce must be single-use and server-issued. Reusing one turns a capture into a login.

---

## 2. Storage — SpaceDB, and only SpaceDB

**Repo:** https://github.com/Remade-With-Rust/spacedb · MIT OR Apache-2.0 · `spacedb-sdk 0.5.2`

Local-first, CRDT-native, mesh-replicated. Your data lives encrypted across machines near your
users — offline-available, converging automatically, every access gated by a signed, scoped,
revocable capability (for humans **and** AI agents). No connection string, no server, no network
required to start.

```toml
[dependencies]
spacedb-sdk = "0.5"          # composes the whole stack; installs rusty_alloc by default
```

```rust
use spacedb_sdk::{
    Database, Schema, CrdtType, Tier, Identity, Capability, SignedCapability,
    Scope, Ops, Outcome, StrongResult,
};

// 1. Open an offline-first local replica for this device.
let mut db = Database::open(Identity::generate("did:mata:home-1")?);

// 2. Each field picks its CRDT type AND its consistency tier.
db.define(
    Schema::new("profile")
        .field("bio",          CrdtType::Text,     Tier::Convergent) // auto-merges
        .field("display_name", CrdtType::Register, Tier::Convergent)
        .field("cursor",       CrdtType::Register, Tier::Causal)     // read-your-writes
        .field("visits",       CrdtType::Counter,  Tier::Convergent)
        .field("username",     CrdtType::Register, Tier::Strong),    // globally unique
);

// 3. Grant a capability — to a person or an AI agent — scoped, expiring, revocable, budgeted.
let cap = Capability::grant(
        owner.did().clone(),
        "did:agent:assistant",
        Scope::Collection("profile".into()),
        Ops::READ | Ops::WRITE,
    )?
    .with_expiry(1_702_592_000)
    .with_budget(1_000_000);                  // micro-$MATA it may spend
let mut session = db.session(SignedCapability::sign(cap, &owner)?);

// 4. Write offline. Every op returns the consistency it ACTUALLY achieved.
let outcome = db.put_register(&mut session, "profile", "display_name", "Ada")?;
assert_eq!(outcome, Outcome::Local);          // durable here, converging outward

// 5. Strong tier when you mean it: globally unique, or it cleanly refuses.
match db.claim_unique(&mut session, "profile", "username", "ada")? {
    StrongResult::Committed      => { /* yours */ }
    StrongResult::Rejected(_)    => { /* taken */ }
    StrongResult::Unavailable(_) => { /* no quorum right now — never a divergent commit */ }
}
```

**The model in one line:** open → schema → grant → write/read with honest state → strong when
you mean it.

**Pick the tier deliberately.** `Convergent` for anything that can merge (text, counters, sets).
`Causal` for anything a user must see their own write of. `Strong` only for global uniqueness —
it can return `Unavailable` under partition, and that is the feature.

**The layers**, each a seam an operator (disco) fills — the dependency arrow is always
MATA → SpaceDB, never the reverse:

| Crate | Layer | Seam |
|---|---|---|
| `spacedb-store` | L0 encrypted KV, typed tables | `KvEngine`, `KeyProvider` |
| `spacedb-crdt` | L1 convergent docs, reactive queries | — |
| `spacedb-replica` | L2 anti-entropy sync, honest freshness | `Transport` |
| `spacedb-durability` | L2 erasure shards, placement, self-repair | `ShardStore` |
| `spacedb-access` | L5 mID capabilities, delegation, audit | `KeyDirectory` |
| `spacedb-query` / `-vector` | L4 compute-to-data, on-node RAG | redundant placement |
| `spacedb-consistency` | L3 tiers | strong-tier placement |
| `spacedb-meter` | L6 metering, budgets | `Settlement` |

**Library authors must opt out of the allocator:**

```toml
spacedb-sdk = { version = "0.5", default-features = false }
```

Cargo features are additive across the whole graph. A *library* that takes `spacedb-sdk` with
default features installs `rusty_alloc` into every application that depends on it — and any app
that already chose an allocator then fails to build with an error it cannot fix from its own
manifest. Applications decide the allocator; libraries stay out of it.

Hardened node profile: `features = ["secure"]` (guard pages, encrypted free lists; ~4–7 %
instructions, measured).

---

## 3. Media — remade_ffmpeg_rs

**Repo:** https://github.com/Remade-With-Rust/remade_ffmpeg_rs · Apache-2.0 · pre-1.0

Decode, encode, transcode, mux and probe audio/video. A ground-up Rust rebuild of FFmpeg with
no FFI, no copyleft, and zero memory-safety CVEs on the core path by construction.

> **Name-collision trap.** The `rff` crate published on crates.io is an unrelated fuzzy text
> selector. **Always depend by git URL.**

```toml
[dependencies]
rff = { git = "https://github.com/Remade-With-Rust/remade_ffmpeg_rs", rev = "<pin a commit>" }
```

```rust
use rff::{Engine, transcode, probe};

let engine = Engine::new();                 // every built-in codec + container registered
let report = transcode::run(&engine, &spec)?;
```

`rff` is the facade — it re-exports `rff_core` / `rff_codec` / `rff_format` and builds a wired
`Engine`. The `ffmpeg` / `ffprobe` CLIs are thin wrappers over exactly this API; there is no
logic in them you cannot reach programmatically. Codec crates are `rff-codec-*` (aac, av2,
avif, flac, gif, h264, jpeg, jxl, mp3, opus, pcm, png, vorbis, vp9, webp…), containers
`rff-format-*`.

**Use it for:** network and edge media conversion — transcoding uploads, generating thumbnails
and previews, normalising user media before it enters storage, adaptive delivery.

**Honest status.** Conformance is bit-exact where claimed (VP9 315/315 vectors). Speed varies
by codec and is reported as measured: AAC ~6× and Vorbis ~5.3× faster than FFmpeg
(frame-parallel), Opus 1.50–1.60× at quality parity, MP3 decode 1.24× on one core, PNG decode
~2.6×; VP9 decode is ~0.16–0.21× and still optimising. Pick the codec knowing the row.

Sibling single-format crates when you need only one: `rusty_png`, `rusty_jpeg`, `rusty_gif`,
`rusty_flac`, `rusty-opus`, `rusty_h264`, `rusty_av2d`, `rusty_dds`, `rusty-av1-toolkit`.

---

## 4. AI — FFAI, and the two-layer rule

**Repo:** https://github.com/Remade-With-Rust/FFAI · published on crates.io

OCR, ASR/TTS, detection and vision-language in one pure-Rust toolkit. No Python runtime, no
gated weights, no ONNX.

| Component | Crate | Task |
|---|---|---|
| **Mercury** | `ffai-mercury` | ASR + TTS (Whisper/WhisperX-class, VITS/Piper-class) |
| **Carmenta** | `ffai-carmenta` | OCR — documents, screens, change-gated live frames |
| **Diana** | `ffai-diana` | Object detection (YOLO26) + ByteTrack; `ffai-wasm` runs it in a browser |
| **Argus** | `ffai-argus` | VLM captioning / video understanding |
| infra | `ffai-core` (types, engine traits, registry), `ffai-media` (ingest/egress, backed by remade_ffmpeg_rs), `ffai-models` (weight manifests + cache) | |

```toml
[dependencies]
ffai-core   = "0.6"
ffai-diana  = "0.7"     # add only the engines you actually use
```

### The two-layer rule — candle and mistral.rs are not alternatives

mistral.rs is *built on* candle. Picking one does not exclude the other; picking the wrong
**layer** is the actual mistake.

```
mistral.rs     LLM serving — KV-cache paging, GGUF/ISQ, sampling,
     |         grammar-constrained decoding, tool calling
     v
candle         tensor spine — Tensor/Device, ops, model architectures,
               CPU / CUDA / Metal / wasm32 backends
```

| Building | Layer |
|---|---|
| Chat, completion, agentic tool calling | **mistral.rs**, behind one `ChatEngine` seam |
| OCR, ASR/TTS, detection, depth, VLM, embeddings, classifiers, re-rankers | **candle** |
| A model in the browser | **candle** → `wasm32` |
| A tensor op inside either | **candle** — one `Tensor`/`Device` type across all engines |

- **Don't** hand-roll an LLM serving loop on raw candle — paging, quantization, sampling and
  constrained decoding are exactly what mistral.rs already solved.
- **Don't** pull mistral.rs in to run a 20 MB embedding or detection model — that is candle's.
- **`candle-core`, `candle-nn`, `candle-transformers` move in LOCKSTEP.** Mixing minor versions
  is a type mismatch, not a warning. Pin all three once, in the workspace manifest.
- **Never** route vault-adjacent context to a remote API. Local inference is the default; BYO
  cloud keys are an explicit, per-user choice.
- **Never** llama.cpp, Ollama, or any C/C++ inference stack.
- Pair grammar-enforced JSON-schema output with your tool descriptors — constrained decoding is
  what makes small local models emit valid tool calls.

**Two C caveats — state them, never paper over them.** (a) The `cuda`/`metal` features forward
to candle and pull C/CUDA/Metal tooling: a *knowing* exception to the pure-Rust posture. (b)
`candle-core` takes `tokenizers` with `features = ["onig"]` as a hard dependency, so `onig_sys`
compiles a C regex engine into every native candle build — build-time only, no runtime
dependency, target-gated out on `wasm32`, fixable with one feature line upstream. **Never claim
"no C in the tree" for a candle build.**

---

## 5. The allocator — rusty_alloc

**Repo:** https://github.com/Remade-With-Rust/rusty_alloc · MIT · `rusty_alloc` / `rusty_alloc-api`

A pure-Rust rebuild of mimalloc's architecture. No C in the tree, permissive licence, runs on
`wasm32-unknown-unknown` with no emscripten — the last C allocator out.

**Version reality (check before you pin):** the repo is at `1.1.0` with a **frozen API**;
crates.io currently serves `1.0.1` for both `rusty_alloc` and `rusty_alloc-api`. Pin exactly
what you verified. Treat **0.3.2 and earlier as unsound on every target** — 0.4.0 fixed three
platform-independent use-after-frees.

### Two ways to adopt it. Both keep it out of your libraries.

**(a) Your own one-crate seam** — preferred for a workspace with several deliverables. The seam
holds the exact pin, the startup configuration and the `secure` feature, so feature code never
names the allocator crate:

```toml
# crates/myapp-alloc/Cargo.toml
[dependencies]
rusty_alloc-api = { version = "=1.0.1" }
```

```rust
// crates/myapp-alloc/src/lib.rs
#![no_std]
pub use rusty_alloc_api::RustyAlloc as Alloc;
```

```rust
// crates/myapp-server/src/main.rs   — the DELIVERABLE, exactly once
#[global_allocator]
static ALLOC: myapp_alloc::Alloc = myapp_alloc::Alloc;
```

**(b) `rusty_alloc_default`** — the org's ready-made seam (`0.1`), already the default
`rusty-alloc` feature of `rusty_symbols` / `rusty_tokens` / `rusty_a11y` so those can each
default-on without fighting (one link, one allocator):

```toml
rusty_alloc_default = "0.1"
# hardened: rusty_alloc_default = { version = "0.1", features = ["secure"] }
```

If your app already installs an allocator, set `default-features = false` on the UI crates.

### The law

**Declare it exactly once, in the deliverable — never in a library.** A program may define
exactly one `#[global_allocator]`. A library that declares it forces the choice on every
consumer and makes two such libraries impossible to link into one program. The allocator is a
property of the *deliverable*, not of a component.

✅ Every deliverable: desktop/mobile/web entry points, WASM bundles, daemons, every service,
every CLI. When the deliverable is a staticlib/cdylib app, it goes in that crate's `lib.rs`.

❌ Any shared library crate. In a diff, that is a stop-the-review finding.

### Adopt it for the safety posture, not for speed

A double free **aborts** instead of putting a block on a free list twice and handing identical
memory to two owners (~0.4 % overhead). **Treat that abort as a bug to fix, never a check to
disable.** On a mesh node you do not own, a memory bug becoming a visible crash instead of
silent divergence is the whole point — a replica that corrupts its own heap is a replica that
lies to the mesh.

The published evidence is **parity**, not superiority: instruction counts at-or-below mimalloc
on real programs, ~2–16 % under jemalloc, ~18 % under glibc. **Never claim a speed or footprint
win without a measurement that clears the noise floor** (`performance.md` §2).

Enable `secure` on services exposed to untrusted input; `debug_checks` in debug profiles only.
Replaces mimalloc / jemalloc / snmalloc (all C/C++) and `dlmalloc` on `wasm32`.

---

## 6. Compression — rusty_zstd

**Repo:** https://github.com/Remade-With-Rust/rusty_zstd · MIT OR Apache-2.0

Pure-Rust Zstandard (RFC 8878). Compress (levels −7…22, all 9 strategies) and decompress, both
directions interop-verified against facebook/zstd v1.5.7. Dictionaries, long-range matching,
seekable frames and multi-threading are dual-gated against the C reference. No C, no FFI on the
core path.

> **Not on crates.io yet** — it is deploying. Depend by git (pin a commit) or path, and switch
> the pin, not the API, when it publishes.

```toml
[dependencies]
rusty_zstd = { git = "https://github.com/Remade-With-Rust/rusty_zstd", rev = "<pin>" }
```

```rust
// one-shot
let packed   = rusty_zstd::compress(&bytes, 3)?;
let restored = rusty_zstd::decompress(&packed)?;

// streaming
use rusty_zstd::{Compressor, Decompressor, Flush};

// dictionaries — the big win on many small similar payloads
let dict = rusty_zstd::train(&samples, &TrainOptions::default())?;
let packed = rusty_zstd::compress_using_dict(&bytes, &dict, 3)?;

// seekable frames — random access without decompressing the whole archive
let packed = rusty_zstd::compress_seekable(&bytes, 3, DEFAULT_FRAME_SIZE)?;
let chunk  = rusty_zstd::decompress_frame_at(&packed, i)?;

// multi-thread for large payloads
let packed = rusty_zstd::compress_mt(&bytes, 3, rusty_zstd::default_nb_workers())?;
```

**Where it pays.** Anything crossing a link you pay for or a disk you replicate: SpaceDB values
before encryption, media segments, static site bundles before `disco sites deploy`, log and
telemetry batches, and any internal RPC payload big enough to notice.

**Order matters: compress, then encrypt.** Ciphertext is incompressible. Compressing *after*
encryption gains nothing and compressing *attacker-influenced* data alongside secrets leaks
length — keep secret and attacker-controlled data in separate frames.

The `no_std` + `alloc` build works for embedded and WASM targets. No speed claim is made against
C zstd yet; do not repeat one that is not in the ledger.

---

## 7. UI chrome — thoth (or the split trio)

**Repos:** `thoth` · `rusty_symbols` · `rusty_tokens` · `rusty_a11y` · MIT · all `no_std`/wasm-checked

Everything a UI needs before you write a component. All four default-install `rusty_alloc`
(opt out with `default-features = false`).

| Crate | Version | What it gives you |
|---|---|---|
| `thoth` | `0.3.0`, git tag `v0.3.0` | all three below, unified: `thoth::symbols` / `::tokens` / `::a11y` |
| `rusty_symbols` | `0.1` crates.io | semantically named Unicode glyph constants + VS15 presentation pinning |
| `rusty_tokens` | `0.2` crates.io | semantic CSS custom-property names, neutral defaults, `:root` sheet emitter |
| `rusty_a11y` | `0.2` crates.io | labelled glyphs, ARIA live regions, status announcements — as HTML string builders, no DOM crate, no JS |

**Why glyph constants and not literals:** application `.rs` files stay ASCII, so a
Windows-1252 round-trip cannot mojibake your icons. One source of truth, `\u{...}` escapes,
presentation pinned so a WebView renders the same glyph on every platform.

**Why tokens and not hex:** token names *are* the CSS API (`--rt-color-fg`). Defaults are a
small neutral starter; apps override in CSS. `css::root_sheet()` injects the `:root` block into
a WebView or Dioxus app.

---

## 8. The rest of the ladder

### Crypto — never roll it

| Concern | Use | Never |
|---|---|---|
| Key derivation | **Argon2id** | PBKDF2, bcrypt, raw hashes, "we'll tune it later" |
| Encryption at rest | **AES-256-GCM**, authenticated, per-entry, 12-byte random IV | ECB/CBC, unauthenticated modes, a reused IV |
| Primitives | **RustCrypto** (https://github.com/rustcrypto) | C-backed openssl / libsodium bindings |
| TLS, system tools | **memorysafety.org** — rustls, sudo-rs, curl-rust | native-tls / openssl-sys where rustls fits |

### The Oxi* replacements — each deletes a C build

| Need | Use | Replaces | Hard limit |
|---|---|---|---|
| Text shaping, layout, rasterization | **OxiText** | FreeType, HarfBuzz | the point is WASM + no native build tools |
| Internal wire format (service↔service, RPC) | **oxicode** | JSON on internal hot paths | ❌ **not for public APIs** — browsers and third parties get JSON / OpenAPI / gRPC |
| High-precision & arbitrary-size math | **OxiNum** | `rug`, `gmp-mpfr-sys`, `num-bigint`/`num-rational` | ❌ **never for crypto, keys or signatures** — not constant-time, leaks via timing on secret data |
| Protobuf codegen | **OxiProto** | `prost-build` + system C++ `protoc` | pure Rust, inline in `cargo build` |

### Supply chain — Deputy

`cargo install deputy-cli` — takes the full transitive closure of your repos, downloads every
crate into a local encrypted vault you own, SHA-256-verifies on acquisition, re-checks on scan,
and gates what reaches production. A re-published `name@version` with different bytes is
flagged, not silently accepted. Stand it up before your first production build, not after your
first incident.

### The org inventory — check it, it moves

https://github.com/orgs/Remade-With-Rust/repositories

Infrastructure: `mid`, `sovereign-id` (JS), `spacedb`, `rusty_alloc`, `rusty_alloc_default`,
`deputy`.
Media: `remade_ffmpeg_rs`, `rusty_png`, `rusty_jpeg`, `rusty_gif`, `rusty_flac`, `rusty-opus`,
`rusty_h264`, `rusty_av2d`, `rusty_av2f`, `rusty-av1-toolkit`, `rusty_dds`.
AI: `FFAI`, `mercury`, `carmenta`, `diana`.
UI: `thoth`, `rusty_symbols`, `rusty_tokens`, `rusty_a11y`.
Apps: `starfire`.
Deploying: `rusty_zstd`.

**Re-check before you build any general-purpose component.** The fastest way:

```sh
gh repo list Remade-With-Rust --limit 100 --json name,description,updatedAt
```

If something on that list does what you were about to write, use it — and if it *almost* does,
open an issue there rather than forking the capability into your app.

---

## 9. The scaffold — copy this

```toml
# Cargo.toml (workspace root) — every pin lives here, exactly once
[workspace]
resolver = "2"
members  = ["crates/*"]

[workspace.dependencies]
# identity + storage
mid-verify  = "0.1"
spacedb-sdk = { version = "0.5", default-features = false }  # libs opt out; deliverable opts in
# allocator (through the seam in crates/myapp-alloc)
rusty_alloc-api = "=1.0.1"
# ui chrome
thoth = { git = "https://github.com/Remade-With-Rust/thoth.git", tag = "v0.3.0" }
# media + compression, by git (see §3 and §6 for why)
rff        = { git = "https://github.com/Remade-With-Rust/remade_ffmpeg_rs", rev = "<pin>" }
rusty_zstd = { git = "https://github.com/Remade-With-Rust/rusty_zstd",       rev = "<pin>" }
# ai — the three candle crates move in lockstep
candle-core         = "0.9"
candle-nn           = "0.9"
candle-transformers = "0.9"

[workspace.lints.rust]
unsafe_code = "deny"        # lift it per-crate, with a SAFETY comment, never workspace-wide
```

```rust
// crates/myapp-server/src/main.rs — a DELIVERABLE
#[global_allocator]
static ALLOC: myapp_alloc::Alloc = myapp_alloc::Alloc;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // LongLived for daemons/services/apps; ShortLived for CLIs and one-shots.
    myapp_alloc::configure(myapp_alloc::Profile::LongLived);
    myapp_server::run().await
}
```

`configure` is what turns **purging** on. Purging is opt-in upstream, and opt-in-*off* is the
configuration whose RSS behaviour is least understood — with it on, a soak held RSS flat. Every
long-lived deliverable calls it.
