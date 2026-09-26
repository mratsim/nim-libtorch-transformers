# nim-libtorch-transformers

A reference snapshot of the [tattletale](https://github.com/mratsim/tattletale) transformers inference stack in Nim, built on libtorch.
This tree is an educational, self-contained copy of the working stack as it stood before its tensor-kernel migration.

It carries the transformers package (models, layers, sampling, stateful KV cache), the libtorch binding it runs on, and the tokenizer.
It also carries the quantized weight formats and the utility packages they import.

Each kept package travels with its own test suite.

This tree is reference and educational material, not an active development tree, and not published as a package.

## Model coverage

The bf16 suites cover the families below with 01 layer internals,
03 full forward to logits, and 04 greedy text generation per family with fixtures.

| family | lab | tested variants |
|---|---|---|
| gemma | Google | 1b, 270m |
| gemma | Google | 12B, 26B-A4B, E2B |
| GLM | Zhipu AI (Z.ai) | GLM-4.7-Flash |
| Kimi | Moonshot AI | Kimi-Linear-48B-A3B |
| Laguna | Poolside.ai | Laguna-XS-2.1 |
| Ling | InclusionAI (Ant Group) | Ling-3.0-tiny |
| Mistral | Mistral AI | Mistral-7B-v0.1 |
| Moonlight | Moonshot AI | Moonlight-16B-A3B |
| North | Cohere | North-Mini-Code-1.0 |
| Qwen3 | Alibaba (Qwen team) | 0.6B |
| Qwen3.5 | Alibaba (Qwen team) | 0.8B |
| Qwen3.6 | Alibaba (Qwen team) | 35B-A3B (MoE) |

The EXL3 suites cover Qwen3-0.6B-EXL3-5bpw with 00 codec, 01 layer internals,
03 full forward, and 04 greedy generation.

## Layout

Packages and facade modules live under `workspace/`, mirroring the upstream monorepo layout.

| directory | role |
|---|---|
| `workspace/transformers/` | model implementations, layers, samplers, stateful KV cache, fixture suites, benches |
| `workspace/libtorch/` | Nim binding over libtorch: Tensor wrapper, NN functional, Python bridge, benches |
| `workspace/toktoktok_tokenizer/` | BPE tokenizer: codec loading, regex pre-tokenization, HF and tiktoken compatibility |
| `workspace/positron_kernels/` | the kernel subset the models import: activations, Hadamard transforms, constants, CUDA static-lib wrapper |
| `workspace/safetensors/` | safetensors reader and writer, including the libtorch tensor adapter |
| `workspace/zstd/` | zstd compression over the system library, used for checkpoint archives |
| `workspace/pcre2/` | vendored PCRE2 10.47 binding for tokenizer pre-tokenization |
| `workspace/data_structures/` | WAVL tree |
| `workspace/bencher/` | benchmarking helpers |
| `workspace/hardware_platforms/` | CPU query, load-time function resolution, CUDA library discovery |
| `workspace/*.nim` | one-line facade modules, the import surface the sources spell as `import workspace/<name>` |

## Build

Requirements come as bullets:

- Nim 2.2 or later with the C++ backend
- PyTorch resolves from a Python venv at the repo root (`.venv`), three directory levels above `workspace/libtorch/vendor`.
- alternatively `nim install_libtorch` fetches a vendor distribution

```bash
nim install_deps          # nimpy, jsony, stew, packedjson@#head, iface
nim install_deps_dev      # zip, chronos (dev-only)
nim install_libtorch      # optional: fetch a vendor libtorch dist instead of the venv
```

The default source mode is `TTT_LIBTORCH_SOURCE=venv`, set it to `vendor` after fetching the distribution.

Compilation always uses `nim cpp`.
`nim check` runs in C mode there and misreports C++ exceptions.

## Tests

All suites run through `config.nims` tasks, from the repo root:

```bash
nim test_libtorch         # tensor wrapper plus raw FFI suites
nim test_safetensors
nim test_toktoktok
nim test_zstd
nim test_transformers     # aggregate, needs checkpoints, see below
nim test_tf_model name=gemma3     # one model's suites (see task help for names)
nim test_tf_kvcache
nim test_tf_harness
nim test_tf_samplers
```

Single file:

```bash
nim cpp -r -d:release --hints:off --warnings:off --passC:"-std=c++20" \
  --outdir:build/tests --nimcache:nimcache/tests \
  workspace/transformers/tests/kvcache/test_kvcache.nim
```

Environment notes:

| environment | effect on tests |
|---|---|
| Python venv | the libtorch Python bridge test (`workspace/libtorch/tests/python_integration/`) needs the venv Python (`uv run`) with torch importable. On hosts without a dynamic libpython on the loader path, the bridge test fails to load Python |
| exllamav3 | the EXL3 quantization test generator imports `exllamav3`, which no pyproject or lock file here declares. It resolves only from the upstream repository environment on a CUDA box. Committed fixtures replay everywhere, regenerating them needs that environment |
| zstd | macOS and Linux link the system zstd by default. Windows builds need the vendored source at zstd 1.5.7: `git submodule update --init workspace/zstd/vendor/zstd`, then `-d:TTT_USE_SYSTEM_ZSTD=false` |
| CUDA | the `workspace/positron_kernels` CUDA paths need `nim make_libpositron_cuda` (nvcc) and a CUDA-capable GPU. The CUDA suites compile only on such a box. Metal runs natively on Apple Silicon |

## Models

Model checkpoints are never committed.
Suites reach them through gitignored paths under `workspace/transformers/tests/hf_models/`.

- each entry there is either a local copy of the checkpoint or a symlink to wherever
  the checkpoint lives on the machine
- the name must match what the test code opens
- the exact placement is machine-local, nothing under `tests/hf_models/` is committed

## Devices

Tests select their device through `TTT_TEST_ON`, with the values `auto`,
`metal`, `cpu`, and `cuda` (see `workspace/transformers/tests/harness/select_device.nim`).

| device | coverage at snapshot time |
|---|---|
| Metal | verified on an M4 Max: tensor suites, kvcache, samplers, layer invariance, tokenizer, safetensors, zstd, and the gemma-3-270m full-forward fixture |
| CPU | the non-model suites and the harness defaults |
| CUDA | the positron static lib and EXL3 fp16 kernel paths compile only on a CUDA box. Suites were compile-gated, not run, at snapshot time |

## License

Dual-licensed under MIT or Apache License 2.0, at your option.
Copyright (c) 2026 Mamy André-Ratsimbazafy.
See LICENSE-MIT and LICENSE-APACHEv2 for the terms.
