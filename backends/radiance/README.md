# Experimental Radiance ABI15 adapter

Start with the [build and deployment guide](../../docs/radiance-integration.md),
then read the [complete performance comparison](../../docs/radiance/PERFORMANCE-COMPARISON.md).
The comparison identifies historical GPU-tested binaries; it does not certify this
new integration on current KVMem master.

`upstream.json` pins the original Radiance revision and SHA256 of the complete native
patch. `tools/prepare_native.py` creates a new isolated checkout; `tools/build_windows.py`
or `tools/build_linux.py` compiles it with the caller's SDK. The source archive also
contains a checksummed offline native tree. Radiance's LICENSE, NOTICE and bundled
third-party licenses must accompany redistributed engine sources and binaries.

The Windows profile uses RX9060XT/gfx1200, budget36864, generation workspace16384,
batch512, context262144, FP8 K/V and MTP3. `tools/serve_windows.py` starts a loopback
OpenAI-compatible service using explicit `windows-profile.json` values and monitors
RAM/commit. No model or platform dependency is installed automatically.

The adapter owns native page references, transactional host spill/restore, recurrent
checkpoints and optional rolling decode. It directly calls current KVMem's pressure
picker; relevance statistics remain legacy mean-key. Cumulative accepted-attention
feedback, RDNA3 and current Radiance ABI31 are **not** claimed by this port.

Latest integration checks: 39 CPU tests and full adapter linkage passed; fresh pinned
native source reconstruction verified. See the integration report for full-native
build status and remaining GPU qualification. Existing historical Linux/DFlash and
Windows/MTP evidence must not be mixed into one configuration.

## Unified deployment entry points

Use `tools/build.py --platform auto|windows|linux` and `tools/serve.py --platform auto|windows|linux`
on the destination OS. Capacity arguments are shared; Windows-specific mapping flags are
not applied on Linux. WSL requires an explicit historically qualified scratch runtime;
see [runtime provenance and rebuild limitations](wsl-runtime/README.md) and
[complete startup examples](../../docs/radiance/CLIENT-QUICKSTART.md).
The compatibility `tools/serve_windows.py` remains available. These entry points do not
switch operating systems or implicitly start WSL. New launcher tests are CPU-only;
latest-core GPU qualification remains a separate release gate.
