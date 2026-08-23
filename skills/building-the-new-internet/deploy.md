# deploy.md — the road to the MATA distributed cloud

**disco** (`disco.mata.network`) is the console: sign in with your mID, browse the service
catalog, provision and pay per use. It is what an AWS console would be if the machines belonged
to the people running them.

This file is what you build against *now* so that launch day is a command, not a migration.

---

## 1. What is actually live — read this before you promise anything

Twelve services. Statuses are honest, not aspirational.

| Service | Status | Like | What it is |
|---|---|---|---|
| **identity** | **Available** | IAM / Cognito | Sign in with mID; the DID *is* the account |
| **hosting** | Preview | Amplify / Vercel | one `sites deploy` seals → places → replicates → serves off the mesh |
| **edge-functions** | Preview | Lambda / Workers | WASM functions, scale-to-zero, metered |
| **object-storage** | Preview | S3 | durable, content-addressed, self-verifying blobs |
| **cdn** | Preview | CloudFront | edge cache + content-addressed serving |
| **bandwidth** | Preview | data transfer | metered egress, ack-gated, bilateral proof |
| **dns** | Preview | Route 53 | owner-gated zones on your domain, GeoDNS |
| **managed-db** | Preview | RDS / Supabase | hosted SpaceDB (CRDT), metered |
| **containers** | Planned | Fargate | microVM compute |
| **gpu-compute** | Planned | EC2 GPU | GPU passthrough |
| **source-control** | Planned | CodeCommit | git over storage + mID auth |
| **settlement** | Internal | billing | Iron Bank — mints and settles `$MATA` for all of the above |

**"Preview" means two different things, and both are true.** The *primitive* layer is built:
each Preview service maps to real crates and the mesh loop closes **provide → prove → PAID** in
isolation. The *developer self-serve* layer is not fully wired: the console still renders some
mock data and several gateway backends are in-memory stubs.

So today disco is a real console shell over an honest catalog of real primitives. Build against
the seams; the switch flips under you.

> **Never ship a claim ahead of the catalog.** If you are writing docs or a landing page, the
> service registry's `status` field is the source of truth, and there are tests enforcing that a
> "live" service names a real backing crate. Copy the status, don't upgrade it.

---

## 2. The five seams — build against these and the deploy is free

The distributed cloud does not ask you to port your app. It asks that you built against seams it
can fill.

| Seam | You build against | disco fills it with |
|---|---|---|
| **Identity** | `mid-verify` — verify locally, no network | the mID roster + KMS resolver |
| **Storage** | `spacedb-sdk` — a local replica | `managed-db`, mesh-replicated |
| **Transport** | SpaceDB's `Transport` trait | iroh + relay across the mesh |
| **Placement** | SpaceDB's `ShardStore` | erasure shards, anti-affinity, self-repair |
| **Settlement** | SpaceDB's `Settlement` trait | Iron Bank, `$MATA` per-use metering |

The dependency arrow is always **operator → your app**, never the reverse. You never depend on a
disco crate. That is the property that keeps your app deployable anywhere, including on your own
hardware, forever.

---

## 3. Identity — the piece that is GA today

Wire this first. It works completely, offline, right now.

```
browser/wallet          your frontend            your backend
     |                        |                        |
     |   1. request sign-in   |   0. issue nonce ------>|
     |<-----------------------|                        |
     | 2. user consents,      |                        |
     |    wallet signs JWT    |                        |
     |----------------------->| 3. POST {jwt}  ------->|
     |                        |                        | 4. verify_mid_response()
     |                        |                        |    -> did, claims
     |                        |                        | 5. check_rollback()
```

Steps 4 and 5 are **pure functions with no I/O**. There is no JWKS fetch, no DID-resolver call,
no `/token` back-channel, no MAU meter. Your backend can be offline and sign-in still works.

`verified.did` is your users-table primary key. See `stack.md` §1 for the full code and the
three traps (nonce reuse, wildcard audience, skipping rollback).

---

## 4. Storage — same API on your laptop and on the mesh

The migration you do not have to do:

```rust
// Today — a local replica on your own box. No server, no network.
let mut db = Database::open(Identity::generate("did:mata:my-node")?);

// Launch day — the same Database, the same schema, the same ops.
// disco fills the Transport / ShardStore / Settlement seams underneath.
```

Nothing above the seam changes. That is the entire point of building on SpaceDB rather than
reaching for Postgres and promising yourself you will migrate later.

**Two things to get right now, because they are expensive later:**

1. **Tier every field deliberately** (`stack.md` §2). `Strong` returns `Unavailable` under
   partition — decide *now* which fields are worth that, because changing a tier changes your
   app's behaviour under a network split, not just its performance.
2. **Per-entry keys, never a blob** (`architecture.md` §3). This one cannot be fixed later
   without a data migration and a merge story.

---

## 5. Shipping a site

### The CLI path

```sh
# 1. pair this machine with your mID (device-auth, roster-verified grant)
disco party
disco whoami

# 2. point at a gateway
export DISCO_GATEWAY=https://<gateway-host>

# 3. deploy a folder
disco sites deploy ./dist --site my-app --domain app.example.com
```

Output tells you exactly what it assembled — site id, file count, byte count, the op path
(`/sites/deploy`), the URL and the required capability. Without `DISCO_GATEWAY` set it assembles
and stops; **sending to the resource gateway is the go-live step**, at which point the gateway
seals → chunks → places → replicates → registers, and the CDN edge serves it off the mesh.

`--site` defaults to the parent directory of a build dir, so `./my-app/dist` becomes `my-app`.
Set it explicitly for anything you will deploy twice.

### The programmatic path — same op, from your own code

```rust
use mata_resource_sdk::{ResourceClient, HttpTransport, PURPOSE_API};

let transport = HttpTransport::new(gateway_url)
    .with_signer(my_did, PURPOSE_API.to_string(), Box::new(device_signer));
let client = ResourceClient::new(transport, caller);

let deployment = client.sites().deploy("my-app", bundle).await?;
let sites      = client.sites().list().await?;
let dash       = client.sites().dashboard().await?;   // health, footprint, delivery, spend
```

`with_signer` runs the **live mID handshake per request**: fetch a single-use nonce from the
gateway, sign it into a `Resource-Assertion`, attach it. Not a cached bearer token — a fresh
proof every call. Use `with_auth` only for a header signed elsewhere, and `Auth::None` only for
health checks against an open local gateway.

### The six service sub-clients

```rust
client.storage().allocate(gb).await?;              client.storage().stat(id).await?;
client.functions().deploy(op).await?;              client.functions().list().await?;
client.dns().claim("example.com").await?;          client.dns().set_record(op).await?;
client.cdn().set_policy(op).await?;                client.cdn().deploy_regions(op).await?;
client.database().provision("main", 3).await?;     client.database().list().await?;
client.sites().deploy("my-app", bundle).await?;    client.sites().dashboard().await?;
```

`ResourceClient::call` checks the capability **client-side first** and fails with
`SdkError::Unauthorized` without a round-trip. The gateway re-checks server-side. Both checks
exist on purpose — the client-side one is for speed and error quality, never for security.

**Test against `MockTransport`**, which records calls and replays canned responses, then prove
the real path against a genuinely bound server (`architecture.md` §7).

---

## 6. Deploy-readiness checklist

Tick these while you build, not the week you launch.

**Identity**
- [ ] Sign-in verifies through `mid-verify` with a server-issued single-use nonce
- [ ] `check_rollback` is called against the last roster version you stored per DID
- [ ] `expected_audience` is your exact origin — no wildcard
- [ ] Authorization is capability-based (`mata-cap`), so an agent DID can be granted the same
      scoped, expiring, revocable access as a person

**Storage**
- [ ] All persistence goes through `spacedb-sdk` — no connection string anywhere
- [ ] Every field has a deliberate `CrdtType` **and** `Tier`
- [ ] Per-entry compound keys; no struct serializes a whole collection into one CRDT value
- [ ] Per-record DEKs with position-bound AAD, and a written rotation path

**Build**
- [ ] `cargo check` passes on every target you ship, **including `wasm32-unknown-unknown`**
- [ ] No `*-sys` crate, no `protoc`, no CMake, no system C library in the tree
- [ ] `#[global_allocator]` appears exactly once per deliverable and never in a library
- [ ] Dependencies vaulted and scanned through Deputy

**Shape**
- [ ] Every capability is an op callable by CLI, test and agent — not only by the UI
- [ ] Primitives speak bytes/CIDs/DIDs/capabilities; product types live in the thin consumer
- [ ] Content-addressed artifacts, hash verified on every hop
- [ ] Static assets compressed with `rusty_zstd` before they are sealed

**Honesty**
- [ ] Every status claim matches the catalog
- [ ] Every performance number in your docs clears the bar in `performance.md` §2
- [ ] No `unwrap()` on a user-reachable path; no "temporary" shim left in

---

## 7. Operating on a mesh — what bites after launch

- **Throttle *mis*-tuning starves legitimate peers** more often than it stops abusers. Tune
  against observed traffic and log the rejections; a throttle you cannot see firing is a
  throttle you cannot debug.
- **Transport auth is per-stream.** A QUIC-authenticated endpoint id is the identity; gate each
  stream, and remember that identity rotation can evade a throttle keyed on it.
- **Stale capability views** are the quiet failure: a node caches a grant that has since been
  revoked. Bound the cache lifetime and re-resolve on any authorization failure.
- **Instrument the boundary before you guess.** In a long remote chain, each fix reveals the
  next blocker — the fastest path is a staged progress signal on the client and a greppable
  marker in the daemon log at each hop. See `workflow.md` §4.
