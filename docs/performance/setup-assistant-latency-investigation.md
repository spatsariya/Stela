# Performance investigation: macOS Setup Assistant latency

Status: investigation baseline; no performance improvement claimed.

## Reproduction report (2026-11-17)

A community tester booted Stela v0.5.0 on:
- AMD Ryzen 7 3700X (8 cores / 16 threads)
- NVIDIA GeForce RTX 3060 (12 GB)
- 64 GB DDR4
- MSI MS-7C56 motherboard
- Kingston DataTraveler 3.0 31 GB installer USB

Stela v0.4.2 reached an Apple boot screen without visible progress. A freshly created v0.5.0 installer advanced through the Apple progress bar, restarted automatically, and reached the macOS Setup Assistant. Individual Setup Assistant screens then reportedly took **5–10 minutes** to advance. These are user-observed timings, not instrumented measurements.

The boot output included:

```text
Opening the translation cache (left on the USB: this PC's memory has no room for it; new translations stay in memory this boot.)
The boot log on the USB could not be saved. The boot goes on without it.
StelaGPU: WindowServer: opened the accelerator (type 0)
```

The translation-cache warning should **not** be interpreted as proof of physical RAM exhaustion: `crates/stela-boot/src/system.rs` emits it when the translation-pack budget is zero or the pack is skipped. `crates/stela-boot-core/src/host_memory.rs` defines the pack floor, share and margin. Determine the actual memory budget and why the pack was skipped before proposing a memory-allocation change.

## Existing instrumentation to reuse

- `crates/stela-engine/src/blocks/outcome.rs`: `RunOutcome` contains `blocks_executed`, `compiled_blocks`, `disk_cache_hits`, and `memory_cache_hits`.
- `crates/stela-boot-core/src/boot_progress.rs`: displays execution-block throughput during boot.
- `crates/stela-boot/src/system.rs`: reports translation-pack load/skip state.
- `crates/stela-boot-core/src/host_memory.rs`: host-memory and pack-budget calculations.

## Measurement plan

1. Record Stela version/commit, host CPU/GPU/RAM, firmware settings, install medium, guest disk, and whether the run is a cold or warm boot.
2. Capture boot-stage timestamps and the first-boot versus subsequent-boot timings.
3. Measure time from clicking Continue to the next responsive Setup Assistant screen, at least three trials where practical.
4. Record cache budget, translation-pack size and load/skip reason, along with compiled blocks and disk/memory cache hits. Keep diagnostic output opt-in and avoid per-block logging in hot paths.
5. Measure disk I/O and GPU shader/pipeline compilation separately before attributing latency to the translator.
6. Compare against the unmodified same-version build on the same hardware. Report median and spread, not only best-case time.

## Safety and acceptance criteria

- Never change guest disk partitions or disable existing memory safety checks as a performance experiment.
- Keep instrumentation off by default or low overhead.
- Verify boot-to-Setup-Assistant still works, and run relevant existing unit/integration tests.
- Claim a speedup only after reproducible before/after measurements.

## Open questions

- Is the guest system stored on the SSD or being served from USB?
- Why does the translation pack have no available budget on a machine with 64 GB installed RAM?
- Can boot logs be captured reliably when the USB reports that saving its log failed?
- Which fraction of Setup Assistant latency is CPU translation, shader compilation, guest I/O, or guest services?
