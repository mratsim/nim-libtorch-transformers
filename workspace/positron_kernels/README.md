# positron_kernels

Kernel library used by the transformers stack.

## Contents

| module                                        | provides                                                          |
| --------------------------------------------- | ----------------------------------------------------------------- |
| `src/kernels/portable/hadamard_transforms.nim` | FWHT-128 incoherence processing for EXL3 quantization             |
| `src/kernels/portable/activations.nim`         | activation op wrappers (silu, tanh-approximate GELU)              |
| `src/kernels/cuda/rms_norm.cu`                 | fp16 RMSNorm CUDA kernel                                          |
| `src/kernels/cuda/hadamard_rotate_128.cu`      | fp16 Hadamard-128 rotation CUDA kernel                            |
| `make_libpositron_cuda.cu`                     | single-translation-unit builder for the CUDA static library       |
| `libpositron_cuda.nim`                         | C ABI wrapper over the built `libpositron_cuda.a`                 |

## Build

The CUDA kernels are built as a static library by the repo task:

```
nim make_libpositron_cuda
```

`nvcc` must be on PATH. `libpositron_cuda.nim` links `build/libpositron_cuda.a`
and raises at import time if the CUDA library flags are unavailable.

The portable kernels are plain Nim, compiled with the rest of the stack.

## Consumers

- `transformers/src/layers/linear.nim` and `lmhead.nim` call the Hadamard
  rotation on the EXL3 fp16 path.
- `transformers/src/quantizations/` uses the RMSNorm kernel and the codecs
  the Hadamard transform supports.

## Status

The snapshot ships the kernels the transformers stack uses. The wider kernel
set (mega kernels, ceramic tile kernels) and the upstream kernel compiler live
in the upstream project.
