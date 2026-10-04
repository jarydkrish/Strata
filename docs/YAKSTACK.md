# YakStack fork qualification

This branch starts from upstream 0.1.39, commit
`6f32ec070f23ced9f50e704d854d775da52591ab`. It targets YakStack's stateless
Responses client on a single RTX 3090 with Qwen3.8-Flash-Next IQ2_XS,
GPU vision, int8 KV streaming and four slots. No production migration is
implied by the changes.

## Forced Responses tools

`required` and named function/custom selection now prefill the actual
assistant tool-call opening. Named selection also restricts the supplied
catalog. The server buffers tool events until the model finishes, verifies
the selected name and arguments against the supplied JSON Schema, and only
then sends executable tool events. A missing, invalid or truncated call
returns `tool_choice_failed` (502 for non-streaming; `response.failed` after
streaming starts). Unknown selections and invalid schemas are rejected before
generation. No arguments are fabricated or repaired.

This is prefix-guided generation plus validation, not a general grammar
decoder. Invalid model output can still fail. Forced calls skip free-form
reasoning and stream validated arguments after generation; automatic tools
and ordinary reasoning keep their existing streaming path. `jsonschema` is
now included in the install requirements. An older install missing it rejects
forced selection rather than silently weakening validation.

Native XML scalar boolean parameters accept both `true`/`false` and
`True`/`False` when the schema declares a boolean. The real model used `True`
in a multi-field probe; previously that became a string and failed validation.
Streamed and final JSON arguments now agree on the boolean value. String fields
retain their text; quoted boolean strings, `yes`, numeric values and malformed
JSON are not coerced into booleans.

Responses `text.format` uses this path with a private schema-bearing tool.
After validation, its model-generated argument object becomes the JSON text
message; the internal tool never appears in client events. Schema prompting
alone failed the conflicting-instruction probe on the real model. This route
still has no general grammar decoder or hidden retry, and can fail explicitly
on invalid or incomplete generation. It also skips free-form reasoning.

## Four-slot memory boundary

The original 700 MiB reserve ran out of VRAM while instantiating a verify
CUDA graph. The engine logged `verify: instantiate: out of memory`, exited,
and interrupted other requests. The fresh reproduction distinguishes this
from a protocol-only failure. Capture is lazy, so startup health does not
exercise every window and active-slot combination.

The qualification profile increases `--vram-reserve-mib` to 2048 while
retaining vision, KV and speculation settings. This leaves less VRAM for the
expert cache. [yakstack-3090.json](yakstack-3090.json) records the four-slot,
262,144-context configuration used on the host; its `/data` paths assume the
same packed weights, tokenizer, MTP and vision projector layout as the trial.
Use an API key when binding beyond loopback. Preserve the llama.cpp container
and restore it after each trial.

## Measured qualification on 2026-10-04

The tested Python commit is `00ae71932a9b128aff934088420ca700bca73e71`;
later documentation commits do not change the runtime. The image is
`sha256:2b4b3dd2ade039d700fc8346e4bbfb0dc7c1cae1ed63708942a5f5bb53d96132`.
It reuses the upstream CUDA 13 engine built for architecture 86. The host has
one 24 GiB RTX 3090, an i9-13900HK, approximately 91 GiB RAM and PCIe Gen4 x8.
Weights are Qwen3.8-Flash-Next IQ2_XS with int8 KV, a 32,768-token GPU-resident
KV window and GPU vision. This differs from YakStack's dense 27B llama.cpp model.

The local Responses, structured-output, parallel, server and security suites
passed 207 tests. Three consecutive live API contract runs passed the required
gates. Both the unpatched 128K server and patched 262K server, with reserve 2048,
completed 80 short continuation requests across four conversations without an
engine failure. The fork also completed 40 continuations across four roughly
36K-token histories, three 46K-token tool workflows, four-way image probes,
and a 228,063-token input retrieval request in 107.137 seconds.
Four 54,238–54,239-token inputs each generated 4,000 actual tokens in a
308.642-second batch. Four 40,292-token inputs, each containing four images,
generated 1,024 actual tokens in a 118.011-second batch. Normal EOS behavior
was retained. Metrics sampled four simultaneous decoding slots. Across the
fork qualification, sampled free VRAM never fell below 1,284 MiB and host
available RAM never fell below 35.98 GiB. Sampling is not a guarantee against
every transient allocation or arbitrary full-context workload.

All 13 API/protocol probes passed in each of the three contract runs. The
four domain smoke probes are separate: the retired-character exact-answer
probe failed on both the fork and restored llama.cpp. A formatting-versus-
semantic diagnostic is required before calling that a model quality regression.

Fixed 320-token output tests with 6.7K-token synthetic inputs, median of three
repetitions, measured 59.19 aggregate end-to-end tokens/s for four cold requests
and 131.72 for four shared-prefix requests. The fresh llama.cpp control measured
35.93 and 88.81 respectively. These include prefill/queueing and compare different
models. The 40-request long-history soak took 414 seconds and some first-token
waits exceeded 100 seconds; short-prompt throughput is not a general latency
guarantee. Broad quality parity and permanent adoption remain separate decisions.
