---
title: "DeepSeek Built Ascend a Whole Separate Toolchain"
date: 2026-10-02T10:44:22+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent"]
summary: "On the morning of September 30, DeepSeek announced on its official WeChat account that it is open-sourcing infrastructure components for Huawei's Ascend compute platform: a TileLang high-level language compiler tool, a high-performance compute library, and a distributed communication library. Their wording was that these correspond one-to-one with the components previously open-sourced for NVIDIA's platform."
cover:
    image: "/images/deepseek-ascend/cover.png"
    alt: "DeepSeek Built Ascend a Whole Separate Toolchain"
    relative: false
---

## First, the accurate version

On the morning of September 30, DeepSeek announced on its official WeChat account that it is open-sourcing infrastructure components for Huawei's Ascend compute platform: a TileLang high-level language compiler tool, a high-performance compute library, and a distributed communication library. Their wording was that these correspond one-to-one with the components previously open-sourced for NVIDIA's platform.

Reuters reported it plainly that day, calling it another signal that Chinese tech companies are deepening cooperation and looking for alternatives to the NVIDIA ecosystem.

> Chinese AI firm DeepSeek said on Wednesday it has partnered with Huawei Technologies to develop programming tools optimised for Huawei's Ascend chips.
Bloomberg's headline was more direct: the Huawei chip tools DeepSeek just showed off could replace NVIDIA's.

I read a lot of the Chinese coverage, and most of it stops at the four characters "matches CUDA." So this post is not a rehash of the announcement. I went through all six repos, then actually installed TileLang on my own Mac to see what it can compile and what it cannot.

## Six repos, all checked

These are the six links from the announcement. The star counts and creation dates I pulled live from the GitHub API:

| Repo | Stars | Created | Language | License |
|---|---|---|---|---|
| tile-ai/tilelang | 8127 | 2024-10 | Python | Yes |
| deepseek-ai/FlashMLA | 13024 | 2025-02 | C++ | MIT |
| deepseek-ai/TileKernels | 1894 | 2026-04 | Python | MIT |
| deepseek-ai/DeepGEMM-Ascend | 452 | 2026-09-29 | C++ | MIT |
| deepseek-ai/DeepSelect | 449 | 2026-09-09 | Cuda | MIT |
| deepseek-ai/DeepEP-Ascend | 200 | 2026-09-30 | C++ | Not declared |

Note the last row. Of the six repos, only DeepEP-Ascend declares no license; the other five are MIT. If you are doing enterprise vendor selection, that cell is worth confirming yourself.

One more thing worth being clear about: not all of the "Ascend support" is in new repos. I went through the commit history of the original repos:

```text
FlashMLA     2026-09-30  Open-source release for Ascend 950 (2026.09.30) (#229)
DeepSelect   2026-09-30  Add Kernels for Huawei Ascend NPU (#23)
TileKernels  2026-09-30  Release tile-kernels v2.0.0
```

FlashMLA and DeepSelect merged the Ascend kernels straight into the mainline. Only DeepGEMM and DeepEP got separate `-Ascend` repos, because the changes were large. So "one-to-one" is not marketing language. It checks out.

## Where it sits relative to the CUDA ecosystem

Here is how I mapped the six pieces:

- TileLang maps to the layer where you write kernels (where CUDA C++ and Triton live)
- DeepGEMM maps to matrix multiplication, the cuBLAS slot
- DeepEP maps to cross-card communication, the NCCL slot
- FlashMLA covers long-context and sparse attention kernels
- TileKernels covers ordinary vector math and memory-access kernels
- DeepSelect covers data filtering

The official line on TileLang: it is simpler to program than CUDA, it significantly improves development efficiency and simplifies code logic, and it loses no hardware performance. They added one key sentence: the TileLang path was already validated on NVIDIA's platform, and it now carries most of the kernel implementations in DeepSeek V4 series model training.

That sentence carries more weight than any benchmark. It means they are not reinventing something on Ascend. They are moving over the thing they already got comfortable with on NVIDIA.

## The numbers they published

Performance data comes from the official announcement and Huawei's contributed piece. I am relaying it as-is, with the caveats stated afterwards:

- Communication: DeepEP's Dispatch reaches 375 GB/s, Combine 347 GB/s; the team says this is close to the hardware ceiling
- Networking: Ascend 950's SuperPoD Flex with the UBL128 scheme, 128 cards in a single layer at 3.2 Tbps scale-up, 256K cards in two layers of scale-out
- Inference: running DeepSeek-V4.1-Flash in EP32 offline mode, 2469 tokens/s per card at TPOT=5ms, 5102 tokens/s per card at TPOT=10ms (128K context, Dspark speculative acceptance rate 0.85)

One warning: these are vendor numbers, measured under offline inference, and they do not include serving scheduling or framework load balancing. I have no Ascend hardware here, so I cannot run this benchmark. I can only tell you that this is what they said.

## I tried TileLang on my own Mac

Reading the news is boring, so I installed it.

```bash
uv venv --python 3.12 /tmp/tl
uv pip install --python /tmp/tl/bin/python tilelang
# tilelang 0.1.15 | torch 2.14.1
```

The install went smoothly, and there is a macOS arm64 wheel. Then I wrote a minimal kernel, 128×128 element-wise addition:

```python
@T.prim_func
def main(A: T.Tensor((M, N), dtype), B: T.Tensor((M, N), dtype), C: T.Tensor((M, N), dtype)):
    with T.Kernel(T.ceildiv(N, block), T.ceildiv(M, block), threads=128) as (bx, by):
        for i, j in T.Parallel(block, block):
            C[by * block + i, bx * block + j] = A[by * block + i, bx * block + j] + B[by * block + i, bx * block + j]
```

Then I compiled it for three targets. The results were tidy:

```text
target = c (CPU)    ✅ compiled, generated 448 characters of C source
                     run output matches reference: True
target = cuda       ❌ module 'tilelang.cuda._ffi_api' has no attribute
                     'AnnotateDeviceBoundTmaCopies'
target = ascend     ❌ AssertionError: Target ascend is not supported
```

![The same kernel, compiled for three targets](/images/deepseek-ascend/three-targets.png)

The Ascend error annoyed me, so I bypassed its detector and stuffed a Target dict in directly. What came back:

```text
ValueError: Target kind "ascend" is not defined.
Target creation from config dict failed: {"kind": "ascend"}
```

Reading the source explains it. Its Ascend availability check is written to require `torch.npu.is_available()` to be true, meaning a real Ascend runtime has to be attached.

The package actually ships Ascend codegen sources and templates (`src/ascend`, `src/tl_templates/ascend/*.h`, plus an npu interchange layer extension), but the runtime in this wheel does not register an Ascend target. So:

This Mac can prove TileLang is a real compiler (the CPU target compiles and runs correctly). It cannot prove the Ascend path.

To genuinely compile for Ascend, DeepGEMM-Ascend's README is rigid about it:

```text
Requirements
- HUAWEI Ascend NPU (developed and validated on the Ascend 950 series)
- CANN 9.20 toolkit providing bin/bisheng, bin/ld.lld
- Torch NPU package torch_npu
- Python 3.10 or higher
```

`pip install . --no-build-isolation`, and then you also need the card. On my path, the missing item is that last line.

## What this means if you build clients

If you write applications (like the AI chat client I have on my desk), nothing changes in the short term. API prices and response latency stay where they are.

What is actually worth watching in the medium term is the word "supply." If the same model code can run on Ascend with a few lines changed, the usable share of domestic compute goes up, and that downward curve on inference cost gets a little steadier. The relevance to you is not saving money today. It is one more variable on next year's quote sheet.

There is a further effect, further out: if TileLang's "write kernels in a high-level language without losing performance" spreads, the cost of adapting kernels for on-device small models comes down too. If you do on-device inference, watch this, not the benchmark numbers.

## My read

First, the genuinely hard part of this release is not writing the kernels. It is making developers not rewrite code per piece of hardware. Six repos and the one-to-one claim is exactly that, compressed into a sentence.

But the `Target kind "ascend" is not defined` from my test shows the other side: right now, the far end of this path is tied to hard version dependencies like CANN 9.20. Change platform, change version, and wherever the link does not hold it will tell you very bluntly.

Second, the direction is clear. NVIDIA's position was never just about fast chips. It is the twenty years of CUDA accumulated around them: libraries, tools, docs, and people's habits. That is the position DeepSeek and Huawei are going after, and they are going after it with a language already proven in their own training runs, not starting from zero.

Third, the boundaries need marking. I have no Ascend hardware, so every performance number in this post comes from the vendor. What I verified myself is three things only: the real state of the six repos, the fact that Ascend kernels did get merged into the original repos' mainlines, and what TileLang can and cannot compile on that Mac.

References:
- DeepSeek official open-source announcement (relayed by ITHome): https://www.ithome.com/1/008/604.htm
- Huawei contributed piece (including performance and networking data): https://www.qbitai.com/2026/09/499263.html
- Reuters report: https://www.reuters.com/world/asia-pacific/deepseek-partners-with-huawei-develop-chip-programming-tools-reducing-reliance-2026-09-30/
- DeepGEMM-Ascend repo: https://github.com/deepseek-ai/DeepGEMM-Ascend
- DeepEP-Ascend repo: https://github.com/deepseek-ai/DeepEP-Ascend
- TileLang: https://github.com/tile-ai/tilelang

Verification boundary: repo star counts, creation dates, and licenses were read live from the GitHub API; the Ascend commit records come from each repo's commit history; the TileLang install and the three-target compile results were run on this machine (macOS arm64, tilelang 0.1.15, torch 2.14.1); performance and bandwidth numbers are all vendor figures — there is no Ascend hardware here, and no benchmark was run.
