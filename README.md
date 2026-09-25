# Strix Alloy WSL2

**Independent extension for Peonist AI's Halogen on Windows: experimental WSL2 support for AMD Strix Halo.** Keep Windows as your everyday desktop while running Qwen 3.8 Flash Next locally in WSL2—no dual boot.

This is an unofficial compatibility layer, **not a replacement AMD driver**, not a 120 GB VRAM unlock, and not affiliated with or endorsed by Peonist, AMD, Microsoft, Qwen or OpenAI. [Upstream explicitly excludes WSL2](https://github.com/peonist-ai/halogen-flash-server/blob/235dbbe143243bd676abe1ede4a57340c750c94e/README.md); our adaptation does not change upstream's support policy.

## What actually works

On one 128 GiB Ryzen AI Max+ 395 / Radeon 8060S Windows PC:

| Test | Result |
|---|---|
| Real **260028-token input**, 262144-position pool | **790.92 prefill / 36.63 decode tok/s**, 32 output tokens |
| Short MTP prompt in the same run | 409.09 prefill / 44.30 decode tok/s, 540 input / 28 output |
| Three simultaneous smaller requests | 3 × (8212 input + 128 output), all answers validated |
| Three full 260k contexts | **Failed guarded startup capacity test**, no token-rate result |
| Full original upstream serving suite, older profile | 60 responses; **41.20 MTP decode tok/s mean** |

Different workloads, not directly comparable averages. The new 260k result is one synthetic repeated-filler qualification sample with short output, not a long-context quality/soak guarantee. The historical 60-response suite used a different memory profile. Cold/warm samples, aborted starts, memory pressure and timing boundaries are retained in the [detailed report](docs/benchmarks/preflight-20260925.md) and [historical suite](docs/benchmarks/upstream-20260924.md).

**Experimental research release:** exact engine/dependency hashes, one qualified memory profile, foreground supervision. The serving window is deliberately bounded to **30–300 seconds after readiness**. This is not yet an always-on server. See [release qualification](docs/release-qualification.md) before installing.

## Why this exists / project family

We use a local Qwen worker alongside cloud models in our ChatGPT/Codex desktop workflow on the same computer. The optional [Codex Local Model Router](https://github.com/MKM-030/codex-local-model-router) connects compatible local servers to the desktop model picker. It stays separate: this repository runs inference; the router integrates clients. It does not run the hosted ChatGPT service locally or reproduce every hosted feature.

| Repository | Backend | Scope |
|---|---|---|
| [strix-alloy](https://github.com/MKM-030/strix-alloy) | Native Windows HIP / llama.cpp | PROJFIX GGUF + compatible MTP draft sidecar |
| **strix-alloy-wsl2** | Windows + WSL2 + DXG + Halogen Flash | HGN checkpoint + quality overlay; this integration |
| [codex-local-model-router](https://github.com/MKM-030/codex-local-model-router) | Client integration | Optional local-worker selection |

We retain the native repository name to avoid breaking links; `strix-alloy-windows` can be considered separately. These are different engines/formats, not kernels switchable by one flag. The historical Halogen 27B server is a separate upstream product and **not a qualified launcher profile here yet**. Running both large backends simultaneously is not qualified. No REV-N dependency is required.

## How it runs under WSL2

1. Windows keeps its signed AMD display driver. WSL2 exposes `/dev/dxg`, Windows `libdxcore`, and a byte-qualified ROCm DXG library.
2. Models live on **native WSL Ext4**, not directly on `/mnt/c` NTFS. A private copy-on-write mapping adapter permits GPU registration without modifying checkpoint files; binds remain read-only.
3. A hybrid adapter copies selected whole weight ranges to HIP device allocations and registers the remainder. Clean file-cache pages are reclaimed only after a verified independent copy.
4. The pinned engine's preflight otherwise charges all trunk weights against host RAM before that placement exists. A hash-gated, process-local bridge substitutes the planner's actual host demand at one audited instruction. **Real available memory, the original 16 GiB floor and comparison remain intact.** Every actual placement must match the plan; mismatches fail closed.
5. Windows-side startup/runtime guards and exact-container cleanup supervise each bounded run. No fake VRAM count or fake `MemAvailable` is used.

Our contribution is the adaptation, setup, supervision and measurements. Peonist supplies the engine/kernels. The engine file on disk is unchanged; the opt-in bridge changes one instruction in that process's memory. **No installed AMD driver binary was patched, firmware flashed, or Secure Boot disabled for this working profile.**

Halogen's ~47.7 GiB N-Gram table is already disk-backed. Adding llama.cpp's `--lazy-mode on` is not another 47.7 GiB saving here. The quality overlay also improves draft projections; it is **not interchangeable** with llama.cpp's MTP draft GGUF. MTP is useful for solo requests, but our concurrent schedule mostly ran with the speculative head off.

## Requirements and settings

A measured profile, not a universal hardware preset:

| Component | Qualified configuration |
|---|---|
| Hardware | Ryzen AI Max+ 395, Radeon 8060S/gfx1151, 128 GiB physical |
| Windows / driver | Windows 11 build 26200.7462; AMD 32.0.31041.1004 / Adrenalin 26.8.1 |
| BIOS on test machine | AMI 3.10; board-specific, not an update recommendation |
| Supported VGM carve | **64 GiB**; Windows sees 68340748288 B (~63.65 GiB) |
| WSL | Ubuntu 24.04; memory **56GB**, 24 processors, 32GB swap |
| WSL reclaim | `autoMemoryReclaim=gradual`, `sparseVhd=true` |
| Storage | Native WSL Ext4 for HGN files and tokenizer assets |
| Tools | GCC 13.3 and Ubuntu `libssl-dev` (OpenSSL development headers) in WSL, Python 3.12+ on Windows, PowerShell 7, Docker Engine in selected WSL distro |
| Dependencies | Pinned official Halogen 0.13.8 image, exact AMD DXG library, Windows `libdxcore` |
| Runtime | Copy cap 48 GiB; prefill block/arena 2048; overlay on; prompt cache off |

Use only your device's supported BIOS/AMD Software VGM control. Save the previous value and know your recovery procedure; a carve change requires a reboot. Setup does **not** change BIOS, drivers, registry, services, WSL settings, pagefile or security settings. Do not copy the native sibling's 96 GiB carve recommendation or native Linux's minimal-carve advice into this WSL profile.

Example manually reviewed `%USERPROFILE%\.wslconfig`:

```ini
[wsl2]
memory=56GB
processors=24
swap=32GB
localhostForwarding=true
[experimental]
autoMemoryReclaim=gradual
sparseVhd=true
```

Applying WSL settings requires stopping its workloads first. Do not shut down an unrelated active session. Unknown configurations are not silently accepted.

## Setup and use

Obtain the official model, matching overlay and tokenizer under their own terms from [Halogen Flash](https://github.com/peonist-ai/halogen-flash-server), on native WSL storage. Install prerequisites yourself; setup does not install OS packages or implicitly download a 124 GB model.

Working DXG bytes are reproducible from an [official AMD wheel](https://stable.repo.amd.com/rocm/core/whl-next/rocm-sdk-core/rocm_sdk_core-10.0.0-py3-none-linux_x86_64.whl). Supply its downloaded Windows path using `-AmdWheel`, or a matching existing Linux library using `-DxgLibrary`. See [dependency hashes](docs/benchmarks/methodology.md#working-dependency-provenance). Do not replace system libraries or install the entire wheel just for this file.

In PowerShell 7 from the cloned repository, replace example paths:

```powershell
# Read-only preflight: no build, model start or configuration write.
.\Install.ps1 -Distribution Ubuntu-24.04 -ModelDirectory /srv/models/flash-next `
  -DxgLibrary /opt/rocm/lib/librocdxg.so.1

# Explicit local build/configuration with the same verified inputs.
.\Install.ps1 -Install -Distribution Ubuntu-24.04 -ModelDirectory /srv/models/flash-next `
  -DxgLibrary /opt/rocm/lib/librocdxg.so.1

# Bounded real-answer checks; one model run at a time.
.\Start.ps1 -Profile Single32k
.\Start.ps1 -Profile Single256k
.\Start.ps1 -Profile Sessions32k

# Foreground local API; stops 300 seconds after readiness.
.\Start.ps1 -Profile Serve32k -ServeSeconds 300
```

Model size/filesystem checks are the default. For a first independent model
identity check, add `-VerifyModelHash` to the explicit `-Install` command; hashing
the full checkpoint can take substantial I/O time. It is not repeated on every
start. Exact adapter/build/environment and DXG hashes remain mandatory.

API: `http://127.0.0.1:8731/v1`, loopback-only and unauthenticated. Do not expose it to a network. Keep the supervising window open. `Serve32k` has one slot and a 32768-position pool; `Single256k` qualifies the larger context but is not an always-on 256k service. `Trace32k` validates placement demand without large weight allocation: it is **not inference success**.

Machine paths, generated binaries and logs stay in ignored local outputs. Stop the foreground launcher before running `Uninstall.ps1`; it targets only manifest-owned generated files, never models, WSL, Docker or drivers. See [rollback/validation](docs/release-qualification.md).

## Benchmarks: comparison, not a leaderboard

Different weights, prompt lengths, cache states and timing boundaries: **non-equivalent workloads**. PROJFIX is a quantization, not an engine.

| Path | Workload | Prefill tok/s | Decode tok/s |
|---|---|---:|---:|
| pwilkin / ilintar, Linux | Publisher pp16384 / separate tg128 | 1204.31 ± 2.31 | 26.28 ± 0.29, serial |
| Strix Alloy / PROJFIX, native Windows | Historical served 16k; separate depth ladder | 1031 at 16k | 31.0–32.8 MTP across 16k–251904 |
| Strix Alloy / PROJFIX, native Windows | Historical 259-token prompt | Not paired | 45.31 warm MTP median |
| WSL2 + Halogen, older profile | Original 60-response suite | Short prompts; not headline prefill | 41.20 MTP mean |
| WSL2 + Halogen, older profile | Original HTTP size sweep | 782.58 at 8185 / 769.84 at 16393 | Different timing boundary |
| WSL2 + Halogen, current profile | 260028 input / 32 output, one sample | **790.920** | **36.626 MTP** |
| Historical 27B WSL experiment | 5 input / 32 output, old runtime | Not qualified | 9.4 serial / 10.3 MTP / 17.1 DFlash2 |

Sources: [native comparison](https://github.com/MKM-030/strix-alloy/blob/main/docs/benchmarks/engine-comparison.md), [pwilkin's notes](https://pwilkin.github.io/strix-halo/), [all original-suite records](docs/benchmarks/upstream-20260924.md), [new responses and timings](docs/benchmarks/preflight-20260925.md). The 27B row is historical, not a requalified backend.

Three smaller clients delivered **384 output tokens in 39.421 s**: 9.741 aggregate output tok/s **including prefill**, with 24636 cumulative input, 25020 combined. Two clients: 256 output in 30.846 s, 8.299 aggregate tok/s. These are not pure decode throughput or three clients each sustaining the solo rate.

## RAM left for Windows

Completed weight operations: **50.433 GB copied + 22.574 GB host-registered = 73.007 GB payload**. At 256k, rounded engine weights + KV + workspace total **79.0 GiB**. Sampled adapter-wide dedicated + shared peak: **92.429 GB**, not precise model residency or proof of 120 GB capacity.

Windows available bottomed at **26.031 GB (24.243 GiB)** during the successful large test, recovering to about 55–58 GB after stop. Margin over the 24 GiB runtime floor was only ~0.261 GB. Three full contexts crossed the 20 GiB startup stop trigger and briefly reached **19.228 GB** free during cleanup, then recovered. Guards are reaction thresholds, not hard reservations. Do not weaken them to advertise a bigger pool.

The owner's earlier Task Manager snapshot—51.5/95.6 system, 31.8 dedicated, 47.5 shared—belongs to the **older 32 GiB carve**. Those overlap, not independent memory banks to add. We retain bytes and distinguish GB from GiB.

The owner reports a fluid Windows desktop. League of Legends was smooth on an earlier configuration; gaming alongside this WSL profile has **not** been tested. No FPS/latency guarantee is claimed.

## Licensing, credits and next work

Only our adapter sources, scripts, profiles, tests and sanitized evidence are distributed. No modified upstream image, proprietary engine/kernel, AMD binary, model weight or private machine dump. The engine stays under [its EULA](https://github.com/peonist-ai/halogen-flash-server/blob/main/LICENSE.md); [trademark policy](https://github.com/peonist-ai/halogen-flash-server/blob/main/TRADEMARKS.md) is separate. Our original code uses [MIT](LICENSE); see [notices](THIRD_PARTY_NOTICES.md).

Next: repeated full-depth benchmarks, sustained guarded service, memory/layout optimization, larger concurrency, and a separately qualified 27B integration. **The larger performance/memory goal is not complete.** WSL extension issues belong here, not as implied upstream support requests.

Credits: [Peonist AI](https://github.com/peonist-ai/halogen-flash-server), [Qwen](https://github.com/QwenLM), [ROCm / TheRock](https://github.com/ROCm/TheRock), Microsoft's WSL/DXG stack, CIRU runtime work, and [pwilkin / ilintar](https://github.com/pwilkin/llama.cpp/tree/strix-halo).
