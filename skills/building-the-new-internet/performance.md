# performance.md — fast, without breaking it or lying about it

Rust is fast. Idiomatic-*looking* Rust routinely leaves large multiples on the table through
unnecessary allocation, repeated work, single-threading, and O(n) container ops.

Order of operations matters more than any individual technique:

```
profile  ->  the five moves (safe)  ->  measure  ->  only then consider unsafe
```

Every step below is skippable except the first and the third.

---

## 1. The five moves — ~90 % of real codebases

The worked example: `top_users(log: &str)` over a 20-million-line CSV. Baseline ≈ 1.5 s.

### Move 1 — stop allocating memory you don't need

Every `.to_string()`, `.clone()`, `.to_owned()`, `format!` or intermediate `.collect()` in a hot
loop is a heap allocation. 20M lines × 2 = 40M allocations doing nothing.

```rust
// BEFORE — owned keys: two allocations per line
let mut counts: HashMap<String, u64> = HashMap::new();
let user = line.split(',').nth(1).unwrap().to_string();   // alloc

// AFTER — the key borrows from `log`; allocate only the 10 strings you return
let mut counts: HashMap<&str, u64> = HashMap::new();
let user = line.split(',').nth(1).unwrap();               // a view, no alloc
```

≈ 1.5 s → ≈ 1 s from deleting allocations alone.

- Prefer `&str` / `&[T]` / `&T` in signatures and loop bodies; push owning to the edges.
- Reuse buffers across iterations (`String::clear` and refill) instead of a fresh allocation.
- Pre-size with `with_capacity` when the count is known.
- `Cow<str>` when a value is *usually* borrowed and occasionally owned.
- Drop throwaway intermediates: `.map(|x| x*x).collect::<Vec<_>>().iter().sum()` → `.map(..).sum()`.

### Move 2 — stop repeating work

A `HashMap` hashes the key on **every** access. `contains_key` → `insert` → `get_mut` hashes the
same key three times.

```rust
*counts.entry(user).or_insert(0) += 1;    // one hash, one slot lookup
```

Another ~12 %. Generalize: `entry` / `or_insert_with` / `and_modify` for any read-then-write;
hoist loop-invariant computation (regex compilation, config lookups) out of the loop; watch for
two passes where one would do.

### Move 3 — use every core (Rayon)

If iterations are independent, a `for` loop is pinning you to one core.

```rust
use rayon::prelude::*;

let counts: HashMap<&str, u64> = log
    .par_lines()
    .fold(HashMap::new, |mut map, line| {           // per-thread accumulator
        *map.entry(line.split(',').nth(1).unwrap()).or_insert(0) += 1;
        map
    })
    .reduce(HashMap::new, |mut a, b| {              // merge the private maps
        for (k, v) in b { *a.entry(k).or_insert(0) += v; }
        a
    });
```

Down to ≈ 90 ms — **>7×** vs baseline. **Private-then-merge, never shared-and-locked**;
contention on a `Mutex` can erase the entire win.

> **The #1 Rayon compile error:** its `fold`/`reduce` take an identity *closure* (`HashMap::new`),
> not an identity *value* like `std`'s `Iterator::fold`.

Parallelism has fixed overhead. It loses on small N — consider a size threshold with a
sequential fallback, and measure both sides of it.

### Move 4 — pick the right complexity

`Vec::remove(i)` shifts every later element: O(n) per call, O(n²) across a sweep. Removing one
session near the front of a 1M-element vec shifts ~999,999 elements.

- `retain(|s| !s.is_idle())` — one O(n) compacting pass, order preserved. **Usually what you
  want** for bulk filtering.
- `swap_remove(i)` — O(1), order destroyed. The sharp tool for removing *one* element by index in
  a hot path where order is meaningless (a pool, a free list, an entity soup). Don't advance `i`
  afterwards — re-check the element swapped in.

Measured >7,000× on that sweep, from a single method name. **The largest wins are almost always
algorithmic, not constant-factor.** `Vec::contains` is O(n) — use a `HashSet`. A linear scan
inside a loop is an accidental O(n²).

### Move 5 — don't hand-roll what a kernel already does, then check it's called

| you wrote | use instead | why |
|---|---|---|
| `for x in s.iter_mut() { *x = v }` | `s.fill(v)` | broadcast + wide stores — a `u16` fill goes 16 elements per 5 instructions |
| `for i in 0..n { d[i] = s[i] }` | `d.copy_from_slice(s)` | lowers to `memcpy`, dispatched to the widest available width |
| `while a[i] == b[i] { i += 1 }` | a common-prefix helper | `pcmpeqb` + movemask + `tzcnt`, ~16× over byte-at-a-time |
| a hand-rolled checksum byte loop | the crate's kernel | CRC32 has a carry-less-multiply form the compiler cannot derive |
| `v.iter().position(|&b| b == x)` on bytes | `memchr` | same |

**Then verify the fast path is actually reached.** This is the defect that costs most, and every
correctness gate passes it: in rusty_zstd a tested, benchmarked AVX2 checksum kernel carrying a
documented 1.14–1.26× was reachable only from one unit test and one benchmark. The shipping
encoder, decoder and streaming API all took the scalar route — the decode side ran the kernel on
**0 %** of its bytes, for months. The two paths agree by design, so no test could see it.

> **A fast path nothing calls is slow code with extra maintenance.**

**Settle it with a counter, not by reading the call graph.** Two atomics, tally bytes down each
path, run the real workload, print the percentage. One run, no benchmarking rig, no noise floor.
Anything under 100 % is a bug.

---

## 2. The measurement bar — what makes a number admissible

**No claim without a number that clears this bar.** This is not pedantry; it is how you avoid
shipping a "1.6× win" that was your profiler hashing its own output.

- **`--release`, always.** Debug builds are 10–100× slower and lie about *where* the time goes.
- **Profile before you touch anything** (`cargo flamegraph`, `samply`, `perf`). Optimizing cold
  code is wasted effort — and the "obviously hottest" site is regularly not on the shipping path
  at all.
- **Baseline → change one thing → re-measure.** Complexity you cannot justify with a number is a
  regression in disguise. Revert what doesn't pay.
- **Both arms must do identical work.** The single commonest source of a false result. If one arm
  writes 582 MB to disk and the other writes to a null sink, you measured the disk. If your
  profiler hashes every output sample, you measured your profiler — that was 17 % of its own
  runtime in a real case, and it inverted the verdict.
- **Pin the process, interleave the arms (ABBA), N ≥ 31, report median *and* minimum.**
- **Run a null arm** — measure A against A. Whatever separation that shows is your noise floor,
  and any real effect must clear it.
- **Prefer a deterministic count to a duration.** A count, ratio, size or checksum needs no
  pinning and no z-score, refutes bad levers before you build them, and is how you catch the
  instrument lying.
- **Check the reference's defaults.** A comparison against a tool silently using two cores when
  you asked for one is not a comparison.
- **Correctness gate first.** Byte-identical for integer work; tolerance plus SNR for float. A
  speed win that changes output is not a win, it is a bug with good timing.

> Applied to house crates: rusty_alloc's published evidence is **parity**, not superiority.
> rusty_zstd makes **no speed claim** against C zstd yet. Do not repeat a claim that is not in
> the ledger — including in your own README.

---

## 3. When `unsafe` is earned — and the two checks that usually make it unnecessary

An `unsafe` backlog is a list of **hypotheses about generated code**. Most of them are wrong.

### Check one: count the bounds checks the compiler actually emitted

```sh
cargo rustc --release -p <crate> --lib -- --emit asm
# attribute `panic_bounds_check` call sites to symbols by line range
```

Audited that way, a codec's five highest-priority "P1" entries came out:

| entry | emitted checks | verdict |
|---|---:|---|
| "densest bounds-check-per-byte site in the crate" | 14 + 12 — genuinely the most | **0 executions** — not on the shipping path |
| a masked index, "the check is pure tax" | **0** | premise right, conclusion inverted — *because* the masks prove it, LLVM already removed the check |
| scalar quantize | **0** | and the SIMD twin owns the path anyway |
| block extraction | 23 | **the only real one — 13 % of encode** |

Two failure modes, by name: **the code does not run on the path you ship**, and **the compiler
already did it**.

And when a real one survives, **the fix is often still safe Rust**. That 23-check entry was
per-sample `.min()` clamps, dead on every block (the buffer is padded) but blocking the compiler
from proving the index. Hoisting the edge test to block level turned 256 checks into 2 —
**1.14× encode, byte-identical, no `unsafe`.**

> Reach for `unsafe` when the bound genuinely **cannot** be proven — not when it merely has not
> been.

### Check two: does a safe kernel already exist, and does production reach it?

See Move 5. Replacing a hand-rolled `unsafe` store loop with safe `fill` has measured 16 elements
per 5 instructions *and* deleted the unsafe block.

### The discipline, when you do write it

`unsafe` does not mean "bad code". It means **the compiler cannot prove this is sound, so you are
vouching** — and you have to be right, because a wrong `unsafe` is real UB, not a lint.

The pattern to imitate: **one small, carefully-checked `unsafe` line inside an interface that is
completely safe to use.**

- **Minimize the surface** — the smallest possible block, never a whole `unsafe fn` body.
- **Name the invariant** in a `// SAFETY:` comment. *If you cannot state it, you cannot uphold
  it.*
- **Expose a safe API.** Callers never write `unsafe` themselves.
- **Keep the safe twin as the oracle**, and gate the two against each other on every change.
- Set `unsafe_code = "deny"` at the workspace and lift it per-crate, deliberately.

The patterns worth knowing when the proof is real: `get_unchecked` / `get_unchecked_mut` for
indices already proven in range, uninitialized buffers with `set_len`, `TrustedLen` for exact
size hints, raw-pointer sharing across threads, and zero-copy reinterpretation. Each needs the
invariant written down.

---

## 4. Why the rules are shaped this way — the memory model in one page

Three independent questions collapse onto one rule set:

1. **Who frees?** → ownership.
2. **How is memory shared?** → **aliasing XOR mutation**.
3. **How is invalid memory prevented?** → no null; `Option<T>` instead.

The elegance is that rule 2 does double duty: extended across threads it answers *no data races*
for free. **Dangling references and data races are the same bug** — aliased mutation — seen at
single-thread and multi-thread scope. That is why `Send`/`Sync` fall out of the borrow checker
rather than being bolted on. And every check is at compile time, so safety costs nothing at
runtime.

### Three principles that generalize beyond Rust

1. **Push checks from runtime to compile time** (or from later to earlier). A whole bug class
   disappears when the check runs before the code can.
2. **Find the one invariant that kills several bug classes at once.** State it explicitly, then
   check each change against it.
3. **Make illegal states unrepresentable in the type system**, not guarded at runtime.

### Two techniques that implement principle 3

**Parse-constructor** — validate once at construction, then carry a type that is provably valid
everywhere downstream:

```rust
pub struct Email(String);          // private field: outsiders cannot build an invalid Email
impl Email {
    pub fn parse(raw: &str) -> Result<Self, EmailError> { /* the ONLY way to make one */ }
}
```

**Type-state** — model each state of a workflow as its own type. A transition **consumes
`self`**, so the old state is gone; a method valid in one state exists *only* on that type:

```rust
impl User<Unverified> { pub fn verify(self) -> User<Verified> { /* ... */ } }
impl User<Verified>   { pub fn send_email(&self, body: &str) { /* ... */ } }
```

An unverified user literally has no `send_email` method — zero runtime checks, and no
`if user.is_verified` guard for a caller to forget. That is the class of bug a `bool is_verified`
field creates and this pattern deletes.

**Use this on the stack in this skill:** capability-scoped sessions, sealed records, typed
amounts, `Outcome::Local` vs `Committed` — all of it is principle 3 applied.

---

## 5. Async, when you have to look inside it

Most app code never needs this. When you are reading or hand-rolling a `Future`, one causal chain
explains why four hard features show up at once:

```
async fn  =>  Future  =>  poll(Pin<&mut Self>, Context<'_>)
                            |                     |
                            |                     +-> Context borrows the waker  => lifetime
                            +-> mutating a pinned self needs get_unchecked_mut => unsafe
```

- An `async fn` is sugar for a state machine implementing `Future`. Implementing it by hand needs
  exactly two things: an `Output` type and a `poll` method.
- `poll` takes `&mut Context<'_>`, carrying a reference to the waker. `'_` is the anonymous
  lifetime — "infer the scope" — and that scope is exactly one `poll` call.
- **`Pin` is a compiler-enforced promise that the value never moves in memory.** It exists
  because a future's state machine can hold references *into its own fields*; if it moved, those
  would dangle. Actually mutating it needs `get_unchecked_mut`, which the compiler cannot verify
  keeps Pin's promise — hence the `unsafe` block where you vouch.

You don't memorize four features; you trace one chain. When you next see `Pin<&mut Self>` or
`Context<'_>`, this is what forced each one — and §3's discipline is what to apply to the
`unsafe` line it forces.

---

## 6. Specialist territory

- **Transcendentals in a hot loop** (`exp`, `ln`, `tanh`, `sigmoid`, `erf`, `powf`) can be
  replaced with range-reduced polynomials that vectorize. Worth it for exp/ln/tanh; **not** for
  `sqrt`/`min`/`max`, which are SSE2 baseline and already vectorized. The round-to-integer step
  is where two independent implementations reintroduced the libm call they had just removed.
- **Codec kernels** — the `codec-*` skill suite's discipline takes precedence over this file for
  encoder/decoder work: profile-first routing, reference-oracle gates, revert-if-not-faster, one
  brick per commit.
