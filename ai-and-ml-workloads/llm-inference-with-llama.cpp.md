# LLM Inference with llama.cpp

**LLM Inference with llama.cpp**

llama.cpp is the primary recommended framework for running LLM inference on the CMP 170HX. It is a C/C++ inference engine that supports NVIDIA CUDA acceleration, uses GGUF quantized model formats, and can be compiled with the `-fmad=false` flag to bypass the FMA throttle.

**Why llama.cpp**

* Compiled C/C++ with CUDA — no Python overhead, direct GPU control
* Supports FP16, Q8\_0, Q4\_K\_M, Q5\_K\_M and other quantization levels
* Built-in `llama-bench` tool for systematic benchmarking
* Active development, broad model support (Llama, Mistral, Qwen, Phi, Gemma, etc.)
* Can be built with `-fmad=false` to work around the FMA throttle
* OpenAI-compatible server mode (`llama-server`) for API access

**What the arXiv Paper Found**

The 2025 arXiv paper (Xing Kangwei, 2505.03782) tested the CMP 170HX with llama-bench and found:

* FP32 inference: severely impacted by FMA throttle — low tokens/second
* **FP16 inference: over 3× improvement versus FP32** — the primary viable inference path
* Decode speed (token generation): memory-bandwidth bound — the CMP 170HX's strongest attribute
* Prefill speed (prompt processing): more compute-bound — more impacted by throttling
* Power efficiency (tokens/W) at FP16: competitive with consumer cards in its price range

The key finding is that FP16 quantization largely sidesteps the FMA throttle because FP16 operations take a different hardware path, and the decode phase of LLM inference is highly memory-bandwidth bound — exactly where the CMP 170HX excels.

**Installation**

**Step 1 — Install prerequisites**

bash

```bash
sudo apt update
sudo apt install -y git cmake build-essential ninja-build
# Verify CUDA toolkit is installed
nvcc --version
# Should show CUDA 12.x
```

**Step 2 — Clone and build llama.cpp with FMA disabled**

bash

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

cmake -B build \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_FLAGS="-fmad=false" \
  -DCMAKE_CUDA_ARCHITECTURES=80 \
  -DLLAMA_CURL=ON

cmake --build build -j$(nproc)
```

`-DCMAKE_CUDA_ARCHITECTURES=80` — targets GA100 (Compute Capability 8.0) exactly, avoiding unnecessary compilation for other architectures and reducing build time.

`-DCMAKE_CUDA_FLAGS="-fmad=false"` — disables FMA generation in all CUDA kernels, applying the workaround globally.

**Step 3 — Download a model**

**Note:** This guide assumes the standard 8GB CMP 170HX variant. 10GB and 16GB variants exist and can accommodate larger models proportionally. All memory estimates below are for the 8GB variant.

Models must fit within available VRAM. Recommended sizes for full GPU offload on 8GB variant:

| Model             | Quantization | VRAM Usage | Recommended?                       |
| ----------------- | ------------ | ---------- | ---------------------------------- |
| Llama 3.2 3B      | Q8\_0        | \~3.5 GB   | ✅ Yes — fast, fits easily          |
| Llama 3.1 8B      | Q4\_K\_M     | \~5.5 GB   | ✅ Yes — good quality/speed balance |
| Llama 3.1 8B      | Q8\_0        | \~9 GB     | ❌ Exceeds 8GB                      |
| Mistral 7B        | Q4\_K\_M     | \~4.8 GB   | ✅ Yes                              |
| Mistral 7B        | Q5\_K\_M     | \~5.7 GB   | ✅ Yes                              |
| Qwen2.5 7B        | Q4\_K\_M     | \~5.0 GB   | ✅ Yes                              |
| Phi-3.5 Mini 3.8B | Q5\_K\_M     | \~3.2 GB   | ✅ Yes                              |
| Llama 3.1 70B     | any          | \~35 GB+   | ❌ Far exceeds 8GB                  |

bash

```bash
# Example: download Llama 3.1 8B Q4_K_M from Hugging Face
cd build/bin
wget https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF/resolve/main/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf
```

**Step 4 — Run inference**

bash

```bash
# Basic inference
./llama-cli \
  -m Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
  -ngl 99 \
  -p "Explain the roofline model in GPU computing:" \
  -n 256

# -ngl 99 offloads all 99 layers to GPU — required for full GPU acceleration
```

**Step 5 — Benchmark with llama-bench**

bash

```bash
./llama-bench \
  -m Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
  -ngl 99 \
  -p 512 \
  -n 128 \
  -r 3
```

Output will show:

* `pp` — prompt processing speed (tokens/second, prefill)
* `tg` — token generation speed (tokens/second, decode)

**Step 6 — Run as OpenAI-compatible server**

bash

```bash
./llama-server \
  -m Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
  -ngl 99 \
  --host 0.0.0.0 \
  --port 8080
```

The server exposes a `/v1/chat/completions` endpoint compatible with any OpenAI client library.

**Performance Expectations**

Based on the arXiv paper results and community testing, expected performance at FP16/Q4 quantization on a 7-8B model:

| Metric                   | Expected Range     | Notes                                         |
| ------------------------ | ------------------ | --------------------------------------------- |
| Token generation (tg)    | 15–35 tokens/sec   | Memory-bandwidth bound — CMP 170HX's strength |
| Prompt processing (pp)   | 100–400 tokens/sec | More compute-dependent                        |
| VRAM usage (7B Q4\_K\_M) | \~5.5 GB           | Fits within 8GB                               |
| Power at load            | \~160–180W         | Integer + memory dominant workload            |

Token generation speed is the most user-relevant metric for interactive use. The CMP 170HX's 1,355 GB/s bandwidth makes it competitive here because decode is almost purely memory-bandwidth bound at batch size 1 — the GPU is continuously reading model weights from VRAM to generate each token.

**Memory Strategy (8GB Variant)**

On the standard 8GB variant, VRAM is a real constraint. To maximize what fits:

* Use Q4\_K\_M quantization as the default — best quality/size ratio
* Use Q3\_K\_M for models that are slightly too large at Q4
* Keep context length moderate — KV cache grows with context and consumes VRAM
* Use `--ctx-size 2048` or `--ctx-size 4096` rather than the default 8192 to reduce KV cache VRAM usage
* Do not run other CUDA processes simultaneously

**Benchmarking Your Setup**

After building, run the full benchmark suite and report results to the community:

bash

```bash
./llama-bench \
  -m your_model.gguf \
  -ngl 99 \
  -p 128,512,1024 \
  -n 64,128,256 \
  -r 5 \
  --output md
```

Submit your results with model name, quantization level, driver version, and VBIOS version.
