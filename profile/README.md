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

## In The Wild with 199.828 Active Installs

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

### [rusty_ESP | ESP32 remade in Rust](https://github.com/Remade-With-Rust/rusty_esp_arduino)
> **The Espressif ESP32 / Arduino application portfolio remade as memory-safe Rust packages, so a home's devices belong to its home computer — not to a vendor cloud.**

* **Focus:** Nine independent packages with one dependency direction — [`rusty_esp_core`](https://github.com/Remade-With-Rust/rusty_esp_core) (shared types and the clock / rng / key-value seams), [`rusty_esp_dsp`](https://github.com/Remade-With-Rust/rusty_esp_dsp), [`rusty_esp_image`](https://github.com/Remade-With-Rust/rusty_esp_image) (esp32-camera and esp_jpeg), [`rusty_esp_video`](https://github.com/Remade-With-Rust/rusty_esp_video) (MJPEG, H.264, RTP), [`rusty_esp_audio`](https://github.com/Remade-With-Rust/rusty_esp_audio) (I2S/PDM, Opus, FLAC), [`rusty_esp_signal`](https://github.com/Remade-With-Rust/rusty_esp_signal) (Wi-Fi CSI radar, LoRa, BLE provisioning), [`rusty_esp_mid`](https://github.com/Remade-With-Rust/rusty_esp_mid) (mID on the chip), [`rusty_esp_iroh`](https://github.com/Remade-With-Rust/rusty_esp_iroh) (the iroh mesh on the chip) and [`rusty_esp_arduino`](https://github.com/Remade-With-Rust/rusty_esp_arduino) (the `setup`/`loop` sketch facade). Pure Rust, no C in the application image, `no_std` cores with ESP-IDF and bare-metal backends.
* **MATA Home Computer:** a device mints its own `did:mata` with the key at rest in encrypted NVS, advertises itself on the LAN, and is **adopted** by the home computer with a signed grant — no vendor cloud, no claiming service, no account. Camera, microphone, radar and telemetry reach the home computer over iroh (QUIC with pure-Rust TLS) and nothing else; every radio frame carries an mID signature; firmware updates are maker-signed, written to a second slot, and rolled back by the bootloader if the new image never comes up. Verified on silicon (XIAO ESP32-S3 Sense, AI-Thinker ESP32-CAM): provisioning from a browser over BLE, the camera page, adoption that survives a hard reset, and 721 of 721 media packets to a subscriber over the board's own network.
* **License:** MIT / Apache-2.0 — the 0.1 crates are on [crates.io](https://crates.io/search?q=rusty_esp), early and said so.

### [rusty_RTOS | FreeRTOS remade in Rust](https://github.com/Remade-With-Rust/rusty_rtos_core)
> **FreeRTOS remade in Rust: the kernel, the ports, the heaps and the standard demo tasks, traced against the C kernel.**

* **Focus:** The fixed-priority preemptive scheduler with task notifications, queues, semaphores, mutexes with priority inheritance, software timers, event groups and stream/message buffers, as a pure state machine over a `Port` seam. Handles are generational indices, never pointers; `#![forbid(unsafe_code)]` everywhere except the fenced context switch and vector table of the Cortex-M, RISC-V and Xtensa ports; a C ABI relinks unmodified FreeRTOS programs, and the standard demo tasks pass as the conformance corpus.
* **MATA Home Computer:** the bare-metal half of the same story. A sensor with no room for ESP-IDF runs the Janus `no_std` packages on Kairos and is fronted by an iroh bridge, so the smallest device on the LAN still has a DID, signed frames and an owner. `rusty_rtos_core` is public here and the family is on [crates.io](https://crates.io/search?q=rusty_rtos).
* **License:** MIT / Apache-2.0

### [mID (MATA mID)](https://github.com/Remade-With-Rust/mid)
> **Permissionless, self-issued digital identity ownership for Rust.**

* **Focus:** Verifies sign-in tokens entirely locally with absolute **zero infrastructure calls**. Perfect for high-privacy and decentralization.
* **License:** Dual MIT/Apache-2.0

### [SpaceDB (MATA DB)](https://github.com/Remade-With-Rust/spacedb)
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
