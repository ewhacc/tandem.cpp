# Changelog

Each release's notes as they were written then; numbers are from that release's build and date. Current numbers are in the
[README](README.md), older comparison tables in [docs/results.md](docs/results.md).

## v0.5.0 (2026-11-23)

- **Up to +15% decode when the model is bigger than RAM.** The experts VRAM evicts are copied back to RAM on a second stream
  while the GPU computes, instead of in line with the uploads (Maya-S on 128 GB 25.2 → 29.0 tok/s, on 64 GB 8.6 → 9.4;
  DeepSeek-V4.1 IQ2_XS on 64 GB 12.5 → 13.1).
- **2× decode when the model just misses fitting in RAM.** Once the RAM store holds every expert, the CPU keeps computing its
  share of the misses instead of uploading all of them (Qwen3.8 IQ4_XS on a 16 GB card with 64 GB: 24.7 → 49.5 tok/s).
- **Faster long prompts when most of the model is in RAM:** layer streaming now also runs when part of the model is on the SSD,
  reading each layer's missing experts one layer ahead (Maya-S on 128 GB, 15.7k tokens: 30–40 → 20–26 s).
- Under a memory limit (Docker, systemd), the RAM store is sized to the limit instead of the host's RAM.
## v0.4.0 (2026-11-18)

- **Long prompts faster again: layer streaming.** A prompt batch of more than 1024 tokens uses every expert of every layer,
  so tandem.cpp now copies the whole next layer into VRAM while the current layer computes (the experts in the VRAM cache
  from there, the rest from RAM, in one batched copy per layer), instead of copying each layer's experts after its router has
  run. The copies hide behind the compute. Time to first token (prompt tok/s), the same prompts as in v0.3.0:

  | Prompt | v0.3.0 | **v0.4.0** | Strata / Maya |
  |---|---|---|---|
  | Qwen3.8 IQ3_S + MTP, 15.8k tokens, RTX 5070 Ti 16 GB + 64 GB | 15.1 s (1,047) | **9.3–10.2 s (1,550–1,700)** | Strata 11.7 s (1,354) |
  | Qwen3.8 IQ3_S + MTP, 15.8k tokens, RTX 5090 + 128 GB | 7.4 s (2,140) | **4.4 s (3,590)** | Strata 2.2 s (7,200) |
  | DeepSeek-V4.1 Maya-XS, 15.7k tokens, RTX 5090 + 128 GB | 16.5 s (953) | **8.6 s (1,830)** | Maya 9.5 s (1,656) |
  | DeepSeek-V4.1 Maya-XS, 5.6k tokens, RTX 5090 + 128 GB | 6.5 s (856) | **4.4 s (1,270)** | Maya 7.8 s (713) |

  It needs every expert outside VRAM in pinned RAM (the model fits in RAM, or the RAM store holds the rest) and one GPU;
  a model read from the SSD (DeepSeek-V4.1 IQ2_XS on 64 or 128 GB) keeps the v0.3.0 path, which is faster there. The two
  layer-sized staging buffers are borrowed from the expert cache during the prompt, like the larger batch buffers. Decode is
  unchanged (RTX 5070 Ti, four interleaved cells each: Qwen3.8 72.6 / 73.4 vs 73.5 / 73.6 tok/s with 512-token prompt batches,
  DeepSeek-V4.1 12.3 / 12.2 vs 12.6 / 12.5). Exact. `LLAMA_MOE_STREAM=0` turns it off.
- **8192-token prompt batches for models with large layers**, chosen automatically: with one GPU and more than 2 GiB of experts
  per layer (DeepSeek-V4.1), a long prompt is limited by the PCIe link, and twice the batch copies each layer half as often
  (Maya-XS above: 11.5 → 8.6 s). Qwen3.8's 1 GiB layers keep 4096. `LLAMA_MOE_UBATCH=N` sets it.
- **Warm start** (documented now, works with the faster prompts): `LLAMA_MOE_HOT_SAVE=hot.txt` writes the experts a session
  used when the server exits, and `LLAMA_MOE_HOT_LIST=hot.txt` puts them in VRAM at the next start, so the first request
  does not start from an empty cache.
- Fixed in the VRAM borrowing: a cache split into several groups (Qwen3.8 has 7) could lend too little for 8192-token batches
  and ran out of memory; it now lends from as many groups as needed.

## v0.3.0 (2026-11-16)

- **Prompts 2–6× faster.** With an expert cache, prompts now run in 4096-token batches instead of 512. Each batch has to
  bring every expert it uses into VRAM, so a 16k-token prompt used to load the experts 31 times; now 4 times. The larger
  batch needs more VRAM for its buffers: it **borrows** that from the expert cache while a long prompt runs and gives it back
  for decode, so decode keeps its full cache (the VRAM is remapped in place: nothing is rebuilt, ~10–100 ms per prompt). Exact.
  Time to first token, RTX 5070 Ti 16 GB + 64 GB (i5-12400F), the same prompts for every engine:

  | Prompt | llama.cpp | tandem.cpp before | **tandem.cpp v0.3.0** | Strata v0.2.9 |
  |---|---|---|---|---|
  | Qwen3.8 IQ3_S + MTP, 5.5k tokens | 28.9 s | 12.4 s | **4.1–7.0 s** | 6.6 s |
  | Qwen3.8 IQ3_S + MTP, 15.8k tokens | 81.9 s | 35.1 s | **14.7–15.5 s** | 11.7 s |
  | DeepSeek-V4.1 IQ2_XS, 5.6k tokens | 722 s | 324 s | **53–103 s** | 58 s |
  | DeepSeek-V4.1 IQ2_XS, 15.7k tokens | 2012 s | 885 s | **189–201 s** | 57 s |

  On an RTX 5090 + 128 GB, Qwen3.8 at 15.8k tokens: 19.4 → 7.4 s (Strata 2.2 s). Decode after the prompt is unchanged on the
  5090; on the 16 GB card it is 72.1 vs 73.6 tok/s (four interleaved cells each, ~2%, near the noise). The borrowing needs
  CUDA's virtual memory management; on AMD (ROCm builds have it off) the cache is simply sized for the larger buffers.
  `LLAMA_MOE_UBATCH=0` restores 512-token batches.
- **Project Maya's DeepSeek-V4.1 GGUFs run on tandem.cpp** after a one-time conversion ([`tools/maya_ds41_convert.py`](tools/maya_ds41_convert.py),
  [below](docs/results.md#deepseek-v41-maya-s-and-maya-xs-the-same-weights-on-both-engines)): the same Maya-XS files on both engines, RTX 5090 + 128 GB,
  **tandem.cpp 60.3 / 59.4 tok/s vs Maya's 39.0 / 39.6** (1.5×, both without a drafter; Maya with its DSpark drafter: 54.9).
- Store mode: when the RAM store is a few slots short of holding every expert next to the VRAM cache, it now holds all but
  those few instead of none (`LLAMA_MOE_ARENA_FILL_PART=0`: the old behaviour).

## 2026-11-15 (second update)

- **Fixed: 128 GB of RAM with a model bigger than RAM (e.g. DeepSeek-V4.1) was killed at load or hung the PC.** On the Linux 7.0
  kernels we tested, part of RAM is a region that pinned memory cannot use (9 GiB on a 128 GB PC), but the kernel counts it as free, and
  the RAM store took it. The store now leaves it out and keeps 2 GiB + 1/64 of RAM free. It also switches on only where it
  helps: when it can hold every expert, or when the experts are larger than RAM. DeepSeek-V4.1 on 128 GB + RTX 5090 runs in
  mmap mode again (31.4 tok/s in today's check; the table row is 33.1, Strata 28.0). `LLAMA_MOE_STORE_AUTO=0` is no longer needed.
- **A big card in a small PC: VRAM and RAM hold every expert.** When the VRAM cache plus the RAM store can hold all the
  experts, they are placed at start (as many as fit in VRAM, the rest in RAM) and nothing is read from the SSD after that. Qwen3.8
  IQ3_S + MTP (78 GiB) on a 32 GB PC with an RTX 5090: **147 tok/s, up from 75**, first token 0.45 s;
  Strata 153–158 on the same PC (0.95×).
- **Faster first token in store mode.** Each expert upload's padding was read from the file, an SSD read per expert.

## 2026-11-15

- **Faster GPU work between the routing steps, on NVIDIA and AMD.** Three kernel changes in the shared CUDA/HIP code: the
  routing results come back to the CPU in one copy instead of three, a faster hyper-connection mixing kernel, and fused
  state copies. Qwen3.8 with one user, A/B on the same PC: RX 7900 XTX +2.5%, RTX 5070 Ti +1.3%. The 7900 XTX same-day table below now
  reads 68.6 tok/s (65.6 on 11-12).
  Exact; no setting needed (`LLAMA_MOE_RB_PACK=0` / `GGML_CUDA_CPY_RUN=0` turn two of them off).

## 2026-11-13

- **Big models on small PCs are much faster.** When a model is far larger than your RAM, tandem.cpp now uses VRAM and RAM
  together as one large expert cache, so far less has to come from the SSD. It switches on by itself; nothing to set.
  - Qwen3.8 (78 GiB) on a 32 GB PC with a 16 GB card: **34 tok/s, up from 29**. That's 2.3× llama.cpp and 1.13× Strata,
    with half Strata's SSD reads and a faster first token.
  - DeepSeek-V4.1 (139 GiB of experts) on a 64 GB PC: +7–10%, and **1.8× Strata** on a budget DRAM-less SSD.
  - Models that fit in RAM run as before. 2 GiB of RAM stays free for your other programs.
  - Details in [Results](docs/results.md); `LLAMA_MOE_STORE_AUTO=0` turns it off ([Defaults](docs/tuning.md#defaults-environment-variables)).

## 2026-11-11

- Models larger than RAM: prompt batches read their missed experts ahead with several threads, and misses read from the SSD
  keep the drive busy. DeepSeek-V4.1 on a 64 GB PC with a 16 GB card is now 1.9× the speed of llama.cpp and 1.4× Strata's (the earlier build was slower than llama.cpp).
- A launcher, `tandem-run.py`, that picks the flags for your machine ([Quick start](README.md#quick-start)).
- More GPU memory for the expert cache: the default reserve is 512 MiB (was 1024), and with `LLAMA_MTP_DRAFT_VOCAB` the MTP
  head's full output layer stays in host memory (only the vocabulary subset is on the GPU).
- The CPU path times its i-quant kernels once at startup and keeps the fastest for your CPU (Qwen3.8 IQ3_S: +4.5% on an i7-8700, +1.5% on an i5-14600KF).
- The CPU/upload split leaves a little more to the CPU (factor 2.25 instead of Strata's 1.85: +2.7% on a 14600K, level elsewhere).
- Pinned copies of the experts use huge pages when the RAM allows (a fragmented host lost up to 30% before).
- Fixes from [issue #2](https://github.com/ewhacc/tandem.cpp/issues/2) (2026-11-12): the CPU-expert threads sleep when idle
  (3.3 cores busy at idle before, 0.03 now), and the auto cache size leaves room for the pooled graphs, which grow with the
  context (long prompts on a 12 GB card ran out of memory).
- The CPU threads default (`LLAMA_MOE_CPU_THREADS`) now counts physical cores within the process's CPU set: CPUs without
  SMT or with E-cores got about half before (14600K: 13 threads instead of 9, level on speed; a 6-core/12-thread CPU is unchanged).
- The RAM rules (pinned experts, the per-layer embedding table, read-ahead) also see a container's memory limit (Docker or
  Podman with cgroup v2, or cgroup v1), not only a systemd scope's.
- Qwen3.8 with MTP keeps one recurrent state and replays the accepted tokens instead of a snapshot per draft position:
  315 MiB more for the expert cache, +2.8% on a 16 GB card (exact; `LLAMA_GDN_REPLAY=0` restores the snapshots).
- Fix (Qwen3.8 with MTP): after a speculative step that rejected draft tokens, a following batch of 64 or more tokens (the
  next turn's prompt) on Ampere, Ada or Blackwell GPUs started from the recurrent state that still included the rejected tokens.
