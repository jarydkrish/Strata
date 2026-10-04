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

The next qualification profile increases `--vram-reserve-mib` to 2048 while
retaining the same context, vision, KV and speculation settings. This leaves
less VRAM for the expert cache. It must be measured through contract warmup,
four-way cold/shared performance, continuation, images and large histories;
the configured reserve alone is not evidence of actual runtime headroom.
Keep the preserved llama.cpp container and restore it after each trial.
