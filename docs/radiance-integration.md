# Radiance ABI15 backend: build, deployment and review status

This is an experimental integration on KVMem `89c66980ed885866251414e1897518bdf008d211`.
The engine remains pinned to StillDeadcode/Radiance
`d5b65cddc6c763861377c77692fb7ee5375222ec` (ABI15, Apache-2.0).
It is not a port to current Radiance ABI31, nor a claim of RDNA3 qualification.
The historical hardware is RX9060XT 16 GiB (`gfx1200`, RDNA4).

## What is integrated

- `backends/radiance/adapter`: native page ownership, byte-preserving spill/restore,
  recurrent checkpoints, statistics observation and optional rolling generation.
- A minimal portable `PackedKeyValue` payload kind permits an engine-native packed
  page without pretending it is separate K and V. Existing enum values and split
  K/V behavior remain unchanged. Packed pages reject token strides and mixed planes.
- Rolling pressure calls the **current core** `kvmem::pressure_selection`; the old
  copied function was removed. Checkpoint and optional prefix-boundary fixes are retained.
- `backends/patches/radiance-abi15-kvmem-windows.patch` contains the complete pinned
  native-engine/platform changes, including LLP64 GPU-address width corrections,
  Windows HIP allocation/mapping, observer hooks and shared/MXFP4 output-head support.
  Source comments and upstream LICENSE/NOTICE remain intact.

**Algorithm boundary:** this adapter still feeds legacy mean-key relevance statistics.
It does not feed the new cumulative accepted-attention feedback used by newer backends.
Calling the current pressure picker does not establish full algorithm parity. The latest
checkpoint/graph-recall code is preserved, not replaced with the old KVMem tree.

## Verification and version boundaries

The 2026-10-30 integration passed all 39 registered CPU tests. Five Python tests first
failed because the sandbox TEMP directory denied writes; rerunning those same five
with a workspace TEMP directory passed. The complete observer executable also compiled
and linked against the qualified ABI15 Windows engine. The complete required Windows
deployment was then built from reconstructed native source: engine, observer executable,
all libr4d kernels, Qwen architecture, reference/quantizer plugins and converter/info tools.
Fresh observer and converter `--help` both exit0. No GPU load was started: the
user's existing DSH service was left running and unmodified.

A final repeat of the 16 Radiance/MemoryContract CPU tests passed. A broader rebuild
attempt could not relink two previously passing upstream test executables because the
OS returned permission denied; that environment failure is retained in the evidence.
No security-software settings were changed to suppress it.

The pinned native patch applies cleanly to a fresh upstream worktree and reproduces all
672 source files after CRLF/LF normalization. A fresh configure through `build_windows.py`
also passed, followed by the full required native/kernel build. Model regression on this
latest-core tree remains required before declaring this new tree production-qualified.
Fresh compilation additionally fixed Windows quantizer/tool portability: CRT M_PI,
S_ISDIR, OpenMP nested-team API and the missing Clang compiler-rt CPU feature-query link.
These are not model re-quantization or changes to the GPU math/selection algorithm.

Historical evidence is separate:

- Windows 33-request executable: `53a7d5fa9c4e749c2f5c5aad050ead1d3e6c7d45c598a310ecb3150e9c6277b5`.
- Current rolling/continuation executable: `5296946322110754b37326629344eacb996f36e2a21154136872e66663fd214c`.
- Those results do not automatically qualify the newly linked executable.

See [the complete same-metric comparison](radiance/PERFORMANCE-COMPARISON.md) for
the old CUDA, WSL and Windows results, quality failures and incompatible configurations.

## Prepare sources (Python + Git)

Run from this KVMem checkout. Choose a new directory; the preparation command refuses
to overwrite an existing checkout. It verifies the patch SHA256 before applying it.

```powershell
python backends/radiance/tools/prepare_native.py --destination ../radiance-abi15
```

The source archive also includes the full patched native tree with a per-file SHA256
manifest for offline builds. Both build helpers accept that tree without Git and
validate its source checksums. For a Git-based build, use the pinned checkout plus the patch. Model
weights are separate; no model or SDK binary is embedded in the code patch.

## Native Windows build

Prerequisites: Windows x64, AMD driver and HIP SDK 7.2, Visual Studio Build Tools with
the x64 C++ toolchain/Windows SDK, CMake >=3.24, Ninja, Python3 and Git. Open an **x64
Native Tools developer shell** and ensure CMake/Ninja/Python are on PATH. SDK paths are
arguments, not a developer-specific drive baked into the scripts.

```powershell
python backends/radiance/tools/build.py --platform windows --source ../radiance-abi15 --build ../radiance-build --rocm "C:/Program Files/AMD/ROCm/7.2" --jobs 4
```

The helper deliberately selects `gfx1200` and the same Clang for C and C++; it builds
the engine, KVMem observer executable, required tools and Qwen architecture/kernel
plugins. Results include `radiance-build/bin` and `radiance-build/radiance_home`.
The Windows target set matches the tested GPU deployment. The optional `libavx` CPU
SIMD plugin still has Linux-only affinity/file-I/O code and is not built by the Windows
helper; this package does not claim general Windows CPU-offload/model support.
Do not mix ABI31 plugins, unrelated kernel registries, or architecture DLLs with ABI15.
Preserve the successful executable, `rad_core.dll`, `radiance_home` and their hashes
together when backing up or reverting a deployment.

CPU-only qualification (no model/GPU needed):

```powershell
cmake -S . -B build-cpu -G Ninja -DKVMEM_BUILD_LLAMA=OFF -DKVMEM_BUILD_SERVER_TESTS=OFF -DKVMEM_BUILD_RADIANCE=ON -DRADIANCE_SOURCE_DIR=../radiance-abi15
cmake --build build-cpu
ctest --test-dir build-cpu --output-on-failure
```

## Start a local OpenAI-compatible service

The intended model repository is
[Zeshion/Qwen3.8-27B-Radiance-KVMem-MXFP4](https://modelscope.cn/models/Zeshion/Qwen3.8-27B-Radiance-KVMem-MXFP4).
The private repository requires owner/granted access. Weight upload completed and the
remote SHA256 matches the qualified local file; this does not make the repository public.

After qualifying the newly built executable, launch it with the separate `.rad` model:

```powershell
python backends/radiance/tools/serve.py --platform windows --exe ../radiance-build/bin/kvmem-radiance-observer-probe.exe --home ../radiance-build/radiance_home --model D:/models/target-head-mxfp4-a.rad --rocm "C:/Program Files/AMD/ROCm/7.2"
```

The service binds only to `127.0.0.1:18380`, model name `radiance-local`; API base is
`http://127.0.0.1:18380/v1`. The launcher requires 10 GiB available physical RAM at
startup and 10 GiB available commit, then stops only its own child if RAM or commit
headroom falls below 2 GiB.
Ctrl+C stops that child. It does not install a Windows service or edit DSH.
`--print-command` validates paths and prints the command without loading the model.

`windows-profile.json` records every explicit switch and environment variable:

| Setting | Profile |
|---|---:|
| Selected GPU KV budget | 36864 tokens |
| Generation/verify workspace (`--kvmem-replay-tokens`) | 16384 tokens |
| Batch | 512 |
| Context limit | 262144 |
| K/V storage | FP8 / FP8 |
| Speculative MTP | 3 |
| Selection block | 128 tokens |
| Host archive limit | 9216 MiB |
| Sink/recent | Common automatic allocation (`-1`) |
| GPU headroom | 256 MiB |
| Embedding placement / visual input | RAM / disabled |
| Rolling generation | Opt-in flag enabled in this explicit profile |

These are deployment-profile defaults, not a claim that the executable's bare CLI
defaults are the same. The 9216 MiB host archive is not 9 GiB of GPU KV. Generation
reserve is physical workspace; request `max_tokens` is a separate output limit.
The profile preserves the qualified mapping flags, MTP shared head, early statistics
seal and 64 KiB small-D2H path; optional failed experiments remain unset.

A client such as DSH can register this as an OpenAI-compatible local provider. Use
`reasoning_effort: "xhigh"`, omit a separate `thinking_budget`, and send an explicit
output cap (the interactive profile uses 65536). Backend compatibility with xhigh
does not guarantee a model will finish within any chosen cap. The old 33-request
benchmark used max_tokens512/thinking128; never mix those numbers with xhigh usage.

## Unified OS entry points and Linux / WSL

`tools/build.py --platform auto|windows|linux` delegates to the existing same-OS
build helper. `tools/serve.py` selects the supervised same-OS launcher. Explicit
wrong-OS selections fail; these are not remote/WSL dispatchers. Existing
`serve_windows.py` remains a compatibility entry point. Run from the KVMem root:

```bash
python3 backends/radiance/tools/build.py --platform linux --source ../radiance-native --build ../radiance-build-linux --rocm /opt/rocm-7.2.3 --arch gfx1200 --jobs 2
python3 backends/radiance/tools/serve.py --platform linux --exe ../radiance-build-linux/bin/kvmem-radiance-observer-probe --home ../radiance-build-linux/radiance_home --model /data/models/target-head-mxfp4-a.rad --rocm /opt/rocm-7.2.3
```

Both platforms use the exact same argument list from `windows-profile.json`:
B36864 / generation reserve16384 / batch512 / context262144 / MTP3 / FP8 KV.
Linux receives shared MTP, early statistics seal and OMP2, not Windows allocation
flags. Inherited experimental KVMEM_/RADIANCE_/KV_DIAG_ variables and LD_PRELOAD
are removed. SDK library paths are explicit. Linux ELF/plugin checks require
`kernels/libr4d.so` and `architectures/qwen35_fp8.so`; these names match the native
CMake plugin output. Linux starts only with MemAvailable >=10 GiB, and stops only
its own child below 2 GiB or when child VmSwap exceeds64 MiB. Windows additionally
checks commit. Resource limits never silently shrink capacity.

On WSL (`/dev/dxg`), append `--wsl-runtime-dir /path/to/verified-rocdxg-reserve-v`.
Missing runtime fails before launching. `librocdxg.so.1.2.0` **and both aliases**
`librocdxg.so.1`/`librocdxg.so` must each resolve to bytes with SHA256
`2b2a082ede46324a5e3771b4b14f2850fdb9bd38348abfbeedee83966bbd6088`.
Only the child gets this directory first in LD_LIBRARY_PATH. No system library
is installed/replaced. The [separate runtime patch, provenance and build notes](../backends/radiance/wsl-runtime/README.md)
are included, but no runtime binary is bundled. Rebuilding generally changes the
hash: qualify that build independently before any explicit manifest update, not
by bypassing this check. Bare Linux must not pass this WSL library option.

`--print-command` / `--dry-run` checks paths without launching a model. The unified
launcher's CPU tests cover platform selection, unchanged capacity, Linux environment,
runtime aliases, resource thresholds, child exit races and child-only cleanup.
These checks and a dry-run do not qualify a new Linux/WSL model deployment.

## Remaining release checks

1. Verify model identity and one/five fixed requests on the latest-core executable;
   keep current DSH untouched
   until that succeeds.
2. Exercise cold Host KV recall, not just hot retention; report spill and restore separately.
3. Repeat full 256K/33 qualification if claiming parity with the old Windows table.
4. Review accepted-attention feedback as a separate algorithm change with independent
   quality/recall tests; RDNA3 and Radiance ABI31 are separate ports.

## Subsequent quick GPU validation — 2026-10-31

The earlier no-GPU statements describe the packaging stage. The fresh Windows build has now completed two Host recall and three prefix requests with unchanged capacity. Independent review passes this limited compatibility screen; see radiance/QUICK-VALIDATION-20261031.md. Raw aggregation failure is retained and explained. No new33-run or Linux/WSL model qualification is claimed.

## Subsequent quick GPU validation — 2026-10-31

The earlier no-GPU statements describe the packaging stage. The fresh Windows build has now completed two Host recall and three prefix requests with unchanged capacity. Independent review passes this limited compatibility screen; see radiance/QUICK-VALIDATION-20261031.md. Raw aggregation failure is retained and explained. No new33-run or Linux/WSL model qualification is claimed.
