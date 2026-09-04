# architecture.md — the shape that survives contact with a mesh

Five patterns and two laws. They are not style preferences: each one is what lets the same code
run on a laptop, on a phone, in a browser, and on a node in the distributed cloud that you do
not own — without a rewrite at any step.

---

## 1. General primitive, thin consumer

The governing rule for anything that could be infrastructure.

**Layer A — the primitive.** Speaks only in bytes, content hashes, DIDs and capabilities. No
product types. No UI framework. No template knowledge.

**Layer B — the consumer.** Produces a generic artifact and calls Layer A. Holds *all* the
product-specific knowledge.

> **Litmus:** *could a developer who has never heard of your product use this op?*
> If the answer is no, product logic has leaked into the primitive.

Worked example: a `sites::Deploy` op takes a content-addressed bundle plus config. A "Publish"
button, an admin console and a CLI are three Layer-B consumers of the **same** op. The
orchestrator behind it depends on placement and transport crates — never on the product.

**Enforcement smell:** if writing the primitive tempts you to `use myapp_business::…`, or to
bake in a template assumption, stop. That belongs in Layer B.

**The payoff is not tidiness.** When the last mile lights up — a real session reaching a real
gateway — it lights up *every* consumer at once, because they share one op. Three surfaces, one
integration.

---

## 2. API first, UI second

Build the op, then the surface. Every capability must be callable by **an agent, a CLI, and a
test**. The UI is consumer #1, never the only consumer.

> If a behaviour only exists inside a component's event handler, it isn't built yet.

This is also what makes your app usable by AI agents on day one, which on this network is not a
nice-to-have: capabilities are granted to `did:agent:*` identities the same way they are granted
to people.

### The three-seam control plane

To add a service you write **one module** and register it in a handful of places. The core never
changes.

1. **Contract** — a struct per op declaring `type Output`, `const SERVICE`, `const PATH`,
   `fn required_capability()`.
2. **SDK** — `Client::call::<O>()` checks the capability **client-side** (fail fast, no
   round-trip), serializes, sends over a **`Transport` seam**, deserializes `O::Output`. Ship
   `HttpTransport` for real and `MockTransport` for tests. Add an ergonomic sub-client per
   service (`.sites()`, `.storage()`).
3. **Gateway** — a `ServiceBackend` re-checks the capability **server-side** and runs the op.

> **The 404 trap.** Registering the backend is usually not enough — there is typically a second,
> hand-maintained list (the router's `parse_service()` arm). Forget it and a fully implemented
> backend returns 404 while every test passes. When you add a service, grep for *every* place
> the service name appears.

Every service ships an in-memory backend for tests and a real one for production, behind the
same trait.

### Catalog honesty

Keep one source-of-truth service registry where each entry carries a `status`
(Available / Preview / Planned / Internal) and a `backing` (the implementing crate). **Make
tests enforce it:** a "live" service must name a non-empty backing crate and ship a code sample;
a "planned" service must be unpriced. This is what stops a console card from claiming GA over a
stub. When you flip a service live, the test forces you to have real backing.

---

## 3. The per-entry CRDT law

**Every data type is per-entry storage.** Each password, contact, document, preference, paired
device is *individually* encrypted and synced under its own compound key:

```
"passwords:{uuid}" -> SyncEntry { blob, updated_at, device_id }
```

> **No single-blob formats. Ever.**

**Why it is a law and not a preference.** Two devices editing *different* entries offline must
**both** survive the merge. A single blob makes every offline edit a whole-vault conflict and
silently destroys one side's work. Only edits to the exact same key may conflict.

**Corollary for review:** if a diff introduces a struct that serializes a whole collection into
one value before it enters the CRDT, that is the bug. Stop there.

**Corollary for the model:** model your app's artifact as a *file tree in the database* —
`WorkspaceFile { path, kind, content: Vec<u8> }` — a git-alternative repo. Store **bytes**, so
uploaded images live alongside text, versioned and synced, with no data URIs.

Adding a synced entity means updating the sync lists in lockstep. Keep the test that asserts the
counts match.

---

## 4. Local-first encrypted storage

- **Encrypt per record.** A random DEK per row, wrapped under the vault key (AEAD). Store
  `{id, user_id, wrapped_dek, ciphertext}`. The list key stays plaintext.
- **Bind the AAD to *position*, not just existence.**
  `AAD = namespace ‖ record_id ‖ field ‖ format_version` — not `record_id` alone, or ciphertexts
  can be relocated and swapped. The format-version byte stops a future format being confused
  with an old one under the same key.
- **Per-record DEKs exist so the vault key can rotate.** Rotation becomes "re-wrap N DEKs", not
  "re-encrypt and CRDT-merge the entire corpus". **A crypto root with no rotation path is a
  permanent liability** — re-wrapping under a new passphrase changes the *wrapping*, not the
  *bytes*, and does nothing once the bytes leak.
- **96-bit random GCM nonces** are safe under the ~2³²-per-key birthday bound. Bound the volume
  per key before any high-volume single-key use — per-record DEKs do this for you.
- **Zeroize everything secret** on **every** path, including error returns. A decrypt-failure
  that returns the DEK un-zeroized is a real gap. Redact `Debug`.
- **User-scope every per-account key, and clear all derived state on account switch.** A global
  flag or blob keyed without `user_id` leaks across accounts on a shared device.
- **State your at-rest medium's *actual* guarantee, not the docstring's.** Browser session
  storage is in-memory and trusted-contexts-only, not "encrypted at rest". That is often fine —
  claiming otherwise misleads the next reviewer.

---

## 5. Content-addressing = self-verifying delivery

Objects are identified by their content hash (BLAKE3). **Verify the hash on every hop** — cache
fill, origin pull, cross-node retrieve.

Four properties fall out for free, and they are exactly the properties a mesh needs:

- Wrong content can never be cached or served.
- A lying node is rejected automatically — no reputation system required.
- Retrieval can fail over across replicas, because any replica returning bytes is checked.
- IDs are stable across processes and runs, so a writer and a reader compute the same id
  independently. That makes the CID a perfect shared-registry key.

---

## 6. The trust boundary and thin clients

**The process holding the vault key, the database, or the provider API keys is the trust
boundary.** Everything else is a thin client that relays *intent* — text — and never keys.

A browser bubble, a CLI, a mobile shell: each POSTs `{token, message, files[]}` to a
**localhost-only** server inside the boundary; the boundary runs the privileged turn and returns
the result.

Gate it three ways: **per-session token** baked into the served client, **localhost bind**, and
explicit CORS. Note that `Access-Control-Allow-Origin: *` means *any* page can POST — the token
is the real defence, so it must reach the legitimate client and nothing else.

---

## 7. Identity is the account

There is no separate login. **The DID is the account.**

A signed grant or action is trusted only because an **enrolled device key** signed it.
Verification is: resolve `DID → DID document → find_entry(device_id) → pubkey`, then check the
signature. Roster rotation (dropping a device) is **free revocation** — the resolver returns
`None` and every grant that device signed stops verifying.

**The client side of the same handshake**, so a request proves who sent it: per request,
`POST /nonce {did, purpose}` → envelope → `sign_assertion(&signer, envelope)` → attach
`Authorization: <Scheme> <b64url(json)>`. The nonce is **single-use**, so this is a per-request
handshake, not a cached bearer token. The gateway verifies it into a `Caller` before dispatch.

Design the transport's auth as an enum (`None` / static header / `Signer{did, purpose, signer}`)
so tests and production share one path.

> **Prove the loop against a real bound server** — a `TcpListener` plus a real serve — not an
> in-process one-shot. Only the socket path exercises the signing and nonce round-trip.

---

## 8. Cores are sound; bugs live at the seams

The single highest-value review heuristic here. Crypto and accounting *primitives* are almost
always right. Every serious finding sits at a seam:

**(a) How an identity is *bound*.** A `did:mata:<base58(pubkey)>` is self-certifying — the DID
*is* the key — but only if the verifier **enforces** it. Recovering the genesis pubkey *from the
DID string* is correct. Resolving `DID → roster` from a mutable central table and trusting
whatever roster you find is identity squatting: whoever can write the table owns the identity.
> **Litmus:** *if I have any valid account, can I claim someone else's un-provisioned DID?*

**(b) Where keys come to *rest*** — and whether they can rotate (§4).

**(c) *Uniformity* of a crypto policy across surfaces.** The most common real bug is the same
check applied on one surface and silently omitted on another. A codebase with four ECDSA
verifiers where only one enforces canonical **low-s** admits malleable `(r, n−s)` signatures —
and now disagrees with itself about what a valid signature *is*. `p256`'s `verify()` accepts
both forms; you must reject high-s yourself.
> **Litmus:** grep every `verify_prehash` / `.verify(` and confirm each has a `normalize_s`
> reject beside it. One missing is a finding.
> **Fix once, in the shared primitive** that every surface imports — a single edit clears N
> findings. If you must duplicate a policy, a conformance test is the *floor*, not the guarantee.

**(d) *Operational* edges** — silent decrypt failures, cross-account state, unbounded nonce
stores. An in-memory unbounded nonce store on an unauthenticated endpoint is an OOM *and* a
replay-window-reopens-on-restart bug.

### The signed-assertion wall — verify in this order, fail closed

1. shape → 2. **audience** match → 3. **purpose** match + allow-list → 4. **issuer** match →
5. resolve DID doc → 6. device in roster → 7. ECDSA verify **with low-s reject** and
point-on-curve → 8. **atomically consume the nonce** (`DELETE … WHERE nonce=$1 RETURNING …`,
never GET-then-DELETE) *after* the signature check → 9. **tamper-check**: the stored envelope
must byte-equal the embedded one.

Audience, purpose and issuer live **inside the signed canonical bytes** *and* are re-checked, so
a nonce minted for service A cannot be replayed at service B even with a shared nonce store.
Fold all failures into one opaque client-side error; log the granular reason server-side.

---

## 9. The durable seam — and the test that lies about it

An `async trait` with **two impls**: an `InMemory` one for tests and single-process, and a
durable or networked one for production. The mantra is "same code, only the connection differs".

> **The trap.** An in-process `InMemory` shared via one `Arc` will pass a "two independent
> readers share state" test **without exercising the real cross-process topology at all**. That
> exact test passed while two real gateways could not share an embedded store.

Always add a test that opens two *independent* connections to the real backing — or is honestly
env-gated against a live server — before claiming the gap is closed. See `workflow.md` §3.
