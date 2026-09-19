<div align="center">
  <h1>Remade With Rust</h1>
  <p><strong>Rebuilding the internet in Rust. Ultimate memory safety, blazing performance, and sovereign.</strong></p>
  <p>Powered by <a href="https://mata.network">@matanetwork</a> / mata.network's commitment to a decentralized, local-first ecosystem.</p>
</div>

<p align="center">
  <a href="https://github.com/Remade-With-Rust"><img src="https://img.shields.io/badge/Organization-Remade--With--Rust-orange?style=for-the-badge&logo=github" alt="Org Badge"></a>
  <img src="https://img.shields.io/badge/Built%20With-Rust-black?style=for-the-badge&logo=rust" alt="Built with Rust">
  <img src="https://img.shields.io/badge/Ecosystem-Local--First-blue?style=for-the-badge" alt="Local-First Focus">
</p>

---

## Core Mission

Remake the internet in Rust. Eliminate an entire network attack surface by overhauling 30+ year old software into a modern architecture. 

Identity. Memory. Media. Transport. AI. Robotics. Everything.

Join the rusty revolution.

---

## In The Wild with 199,828 Active Installs

<a href="https://mata.network">Disco Party</a> is a sovereign distributed cloud and digital freedom toolkit by MATA enabling ownership and accessibility of data.

Build the new internet with our <a href="https://drive.google.com/drive/folders/1VjvP1zSJWH1DmkTqL1qEPt3OM8Ub8jPX?usp=drive_link">AI skills for Rust</a>. Create on any platform with security by design.

FREE RAG Converter Online -- <a href="https://RAGconverter.com">RAGconverter.com</a>

---

## Key Projects

Discover our primary open-source initiatives below:

### [ffmpeg](https://github.com/Remade-With-Rust/remade_ffmpeg_rs)
> **A safe, modern, and pure-Rust reimagining of the industry-standard multimedia framework.**

* **Focus:** Demuxing, muxing, and processing media streams natively without relying on vulnerable C bindings. Built from the ground up for modern pipeline safety.
* **License:** MIT / Apache-2.0

### [FFAI](https://github.com/Remade-With-Rust/ffai)
> **A pure-Rust, local-first equivalent of ffmpeg purpose-built for AI media pipelines.**

* **Focus:** High-performance encoding, decoding, and transformation of media streams optimized for AI workloads — entirely memory-safe and free of C dependencies. (Pre-release)

### [Janus — rusty_esp | ESP32 remade in Rust](https://github.com/Remade-With-Rust/rusty_esp_arduino)
> **The Espressif ESP32 / Arduino portfolio remade as memory-safe Rust packages, so a home's devices belong to its home computer — not to a vendor cloud.**

* **Focus:** Nine packages, one dependency direction: [`core`](https://github.com/Remade-With-Rust/rusty_esp_core), [`dsp`](https://github.com/Remade-With-Rust/rusty_esp_dsp), [`image`](https://github.com/Remade-With-Rust/rusty_esp_image), [`video`](https://github.com/Remade-With-Rust/rusty_esp_video), [`audio`](https://github.com/Remade-With-Rust/rusty_esp_audio), [`signal`](https://github.com/Remade-With-Rust/rusty_esp_signal), [`mid`](https://github.com/Remade-With-Rust/rusty_esp_mid), [`iroh`](https://github.com/Remade-With-Rust/rusty_esp_iroh) and [`arduino`](https://github.com/Remade-With-Rust/rusty_esp_arduino). Pure Rust, no C in the application image.
* **MATA Home Computer:** a device mints its own `did:mata`, keeps the key in encrypted NVS, and is **adopted** by a signed grant — no vendor cloud, no claiming service, no account.
* **Verified on silicon:** BLE provisioning from a browser, adoption that survives a hard reset, **721 of 721** media packets over the board's own network, and a maker-signed A/B update booted and kept.
* **License:** MIT / Apache-2.0 — the 0.1 crates are on [crates.io](https://crates.io/search?q=rusty_esp), early and said so.

### [Kairos — rusty_rtos | FreeRTOS remade in Rust](https://github.com/Remade-With-Rust/kairos)
> **FreeRTOS remade in Rust, and proved line by line against the C kernel's own execution trace.**

* **Focus:** The scheduler and its IPC — tasks, queues, semaphores, priority-inheriting mutexes, timers, event groups, stream buffers — as a pure state machine over a `Port` seam. Generational handles, never pointers; `#![forbid(unsafe_code)]` outside the fenced context switch.
* **Also proved this way:** unmodified FreeRTOS demo programs relink through a C ABI and pass their own checkers, and coreMQTT is remade to **all 218 of its 218 functions**.
* **MATA Home Computer:** the bare-metal half. A sensor too small for ESP-IDF runs the Janus `no_std` packages on Kairos behind an iroh bridge — still a DID, signed frames, an owner.
* **License:** MIT / Apache-2.0 — the family is on [crates.io](https://crates.io/search?q=rusty_rtos).


### [MID (MATA sovereign identity)](https://github.com/Remade-With-Rust/mid)
> **Permissionless, self-issued digital identity ownership for Rust.**

* **Focus:** Verifies sign-in tokens entirely locally with absolute **zero infrastructure calls**. Perfect for high-privacy and decentralization.
* **License:** Dual MIT/Apache-2.0

### [SpaceDB (MATA distributed database)](https://github.com/Remade-With-Rust/spacedb)
> **A local-first, CRDT-native, mesh-replicated database built for a world without data centers.**

* **Focus:** Highly concurrent, distributed data storage that runs natively on the edge or locally without needing constant cloud connection.
* **License:** Apache-2.0

### [Starfire](https://github.com/Remade-With-Rust/starfire) and [Comet](https://github.com/Remade-With-Rust/comet)
> **High-performance, native Rust Sunshine-compatible GameStream client.**

* **Focus:** Low-latency PC gaming streaming client optimized natively for Windows and macOS.

---

## Tech Stack & Toolkit

While **Rust** sits at the absolute core of our system programming goals, we use a tailored stack to push our innovations natively across platforms and runtime targets:

- **Languages:** `Rust` 🦀
- **Paradigms:** Conflict-Free Replicated Data Types (CRDTs), Local-First Architecture, Zero-Trust Architecture, Asynchronous I/O.

---

## Contributing & Community

We are staunch believers in open-source collaboration. If you have an obsession with performance, memory safety, or are looking to build tools that prioritize user autonomy:

1. **Fork** any of our repositories.
2. Create your feature branch (`git checkout -b feature/AmazingFeature`).
3. **Commit** your changes (`git commit -m 'Add some AmazingFeature'`).
4. **Push** to the branch (`git push origin feature/AmazingFeature`).
5. Open a **Pull Request**.

---

<div align="center">
  <sub>Built with ❤️ by the Remade-With-Rust core team. Empowering reliable, fast, and local-first software.</sub>
</div>
