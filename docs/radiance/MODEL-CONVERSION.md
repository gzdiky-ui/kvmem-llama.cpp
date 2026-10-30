# Model conversion provenance

The published model is a separate artifact; see its ModelScope model card and SHA256.
The upstream AMD model is already Quark AWQ MXFP4. We did not retrain its backbone.
The local additions use Radiance's existing RTN quantizer, not new GPU quantization kernels.

`backends/radiance/recipes/qwen27-mxfp4-mtp.recipe` is the **baseline** recipe: MTP
projections remain BF16 and the main output head is FP8. Do not confuse it with
`qwen27-six-mtp-mxfp4.recipe`, which adds the six actual MTP projection rules used in
the first local conversion (FP4 E2M1 / groups32 / E8M0 scale).

The second historical conversion reused the first container but took `output.weight`
from the original checkpoint and overrode only that rule:

```sh
rad-convert /path/to/amd-source --home /path/to/radiance-home \
  --max-spec 3 --kv-cache-dtype fp8 --max-tok 512 --max-ctx 262144 \
  --recipe /path/to/kvmem/backends/radiance/recipes/qwen27-six-mtp-mxfp4.recipe \
  --quant 'output.weight=rtn:codes=fp4_e2m1,group=32,scale=e8m0' \
  --reuse /path/to/mtp-mxfp4.rad -o /path/to/target-head-mxfp4-a.rad
```

This records the original conversion approach, not a claim that a fresh Windows
conversion is byte-identical to the historical ROCm7.2.3 Linux conversion. Use a new
output path, run `--plan-only` first, and audit complete payloads with the included
`compare_rad_target.py` / `compare_mtp_quant.py` tools before model qualification.
The original audits found 1033 unchanged tensors plus six MTP changes, then 1038
unchanged tensors plus one output-head change. Output-head quantization changes logits
and can change draft acceptance/quality; it is not lossless.

The container retains a dedicated U2 MTP output tensor, while the tested deployment
sets `KVMEM_RAD_MTP_HEAD=shared` to use the main head at runtime. That setting is in
the explicit Windows profile. None of these model changes imply K4/V4: the tested
attention cache remains FP8 K and FP8 V.
