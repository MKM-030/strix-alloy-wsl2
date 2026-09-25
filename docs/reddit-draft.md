# Reddit draft — not posted

Suggested title: **Qwen 3.8 Flash Next via Halogen on Windows + WSL2: real 260k input at 791 prefill / 36.6 decode tok/s on Strix Halo (experimental)**

I wanted my 128 GiB Ryzen AI Max+ 395 / Radeon 8060S machine to remain a normal Windows work PC while also running a local Qwen worker. Not dual boot: Windows and WSL2 at the same time.

The result is **Strix Alloy WSL2**, an independent compatibility extension for Peonist AI's Halogen Flash Server:

**Repository:** https://github.com/MKM-030/strix-alloy-wsl2

Important up front: **upstream explicitly does not support WSL2.** This is our experimental adaptation, not an official Halogen Windows release, a replacement AMD driver, or a 120 GB VRAM unlock. Peonist wrote the inference engine and kernels; this project supplies the WSL integration, memory-placement glue, safeguards and measurements.

## What I measured

| Workload | Prefill | Decode | Actual tokens |
|---|---:|---:|---|
| One real ~260k input | **790.92 tok/s** | **36.63 tok/s** | 260028 input, 32 output |
| Short MTP request in the same run | 409.09 tok/s | 44.30 tok/s | 540 input, 28 output |
| Original upstream serving suite, older memory profile | Not a large-prefill test | **41.20 tok/s MTP mean** | 60 responses, 10 prompt shapes, 3 repetitions |

The large test actually occupied 260096/262144 pool positions. It isn't just a server launched with a 256k flag. Prompt caching was off, and real answer/marker checks passed.

**Please keep the caveats:** the large input is repeated synthetic filler, with one measurement and only 32 output tokens. It is not a 260k quality benchmark or sustained-performance claim. The old 60-response suite used a different memory profile, so I am not presenting those rows as an apples-to-apples comparison or cherry-picking the fastest short response as the average. All short cold/warm samples and failures are in the report.

## Parallel requests—and the limit

Three smaller independent requests worked together: each had 8212 input + 128 output tokens. Cumulatively that is **24636 input + 384 output = 25020 tokens**. The group took 39.421 seconds, or **9.741 delivered output tok/s including prefill**. That last figure is not pure aggregate decode speed. Most output tokens ran with the speculative head off during the mixed multi-client schedule.

I also tried **three full 260k contexts**, requiring a 786432-position shared pool. That failed our guarded startup test during KV allocation. Windows free memory crossed the 20 GiB stop trigger and briefly reached 19.23 GB during cleanup. The container stopped and Windows recovered to 57.38 GB available without a reboot. No requests completed, so there is no invented throughput figure for that case.

## What changed technically?

- A byte-qualified ROCm DXG bridge, native WSL Ext4 model storage, and a private copy-on-write mapping adapter for GPU registration.
- Hybrid placement: about **50.43 GB of weight copies to device allocations**, with **22.57 GB host-registered**, totaling **73.01 GB weight payload**.
- An exact-engine-hash-gated, process-local preflight adaptation. The original check charged all trunk weights against host RAM before hybrid placement. It now uses the planner's actual host demand, retaining real available-memory measurements and the original safety floor. The engine file on disk is unchanged.
- Bounded launches, memory guards and exact-container cleanup. No installed AMD driver patch or firmware modification for this working profile.

Halogen's large N-Gram lookup table is already disk-backed; I am not claiming another saving from llama.cpp's lazy-mode flag. MTP and the quality overlay are enabled in the tested solo path; Halogen's overlay is not the native llama.cpp MTP GGUF sidecar.

## Can Windows still be used?

The tested profile uses the supported **64 GiB VGM carve and a 56 GB WSL ceiling**. During the successful large request, Windows available memory bottomed at **26.03 GB**. The engine reported a rounded **79.0 GiB** for weights, KV and workspace; adapter-wide GPU counters peaked at **92.43 GB**. Those measurements overlap and must not be added together or confused with exact physical residency.

The desktop feels fluid to me. League of Legends was smooth with an earlier configuration, but I have **not tested gaming alongside this WSL profile**. The large run also had little margin over the runtime reserve, so this is not a promise that arbitrary extra applications will fit.

## What's in the repository?

Adapter sources, PowerShell setup, guarded launch/rollback, pinned dependency identities and benchmark evidence—not proprietary engine binaries, AMD libraries or model weights. The initial serving mode is deliberately a **finite 30–300-second foreground window**, not an always-on service. Read the release-qualification page before trying it; this remains a narrow one-machine experimental package.

The native Windows HIP/llama.cpp backend stays in [strix-alloy](https://github.com/MKM-030/strix-alloy). This WSL project stays separate, as does the optional [local-model router](https://github.com/MKM-030/codex-local-model-router) used to add a local worker to the desktop model picker. The historical 27B experiment is documented but not yet a qualified backend in this release.

[Full measurements, exact request recipes, response timings and limitations](https://github.com/MKM-030/strix-alloy-wsl2/blob/main/docs/benchmarks/preflight-20260925.md).

Next I want to improve repeatability, longer guarded serving and memory use at depth. Reproducible reports from other Strix Halo systems would be useful—especially with precise driver/image versions, real token counts and clearly separated engine vs. end-to-end timings.
