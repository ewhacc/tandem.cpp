# Results

The [README](../README.md)'s headline table is re-measured on every release. The tables here are history: each says which build
and date it comes from, and they are re-run only when a release changes what they measure.

## Same-day comparison with the current Strata and Maya releases (2026-11-22/23)

Same PC, same day, same model files on every engine, cold start (page cache dropped before every cell), greedy, 20k context,
each engine's own recommended settings (Strata's setup chose its config per PC: its MTP head, 4 drafted tokens, we use 3;
Maya's setup wrote its configs). tandem.cpp: the v0.5.0 code (the 128 GB and 32 GB rows from the dev build of the same day).
The cells use [`bench/run_cell.py`](../bench/run_cell.py) and [`bench/w4_longctx.py`](../bench/w4_longctx.py). The 5070 Ti PC has an i5-12400F, the 5090 PC an i5-14600KF.

Qwen3.8-Flash-Next IQ3_S + MTP head (78 GiB) against Strata v0.2.18, decode tok/s and time to first token on a 5.5k and a 15.7k
token prompt:

| PC | tandem.cpp | Strata | first token 5.5k / 15.7k (ours vs Strata) |
|---|---|---|---|
| RTX 5090 32 GB + 128 GB RAM (model fits in RAM) | 190.5 | **201.1** | 1.8 / 4.5 s vs **0.9 / 2.0 s** |
| RTX 5070 Ti 16 GB + 64 GB RAM | 74.3 | **87.5** | 3.9 / 9.5 s vs **2.3 / 4.2 s** |
| RTX 5070 Ti 16 GB + **32 GB RAM** (model is 2.5× RAM) | **52.4** | 30.2–35.4 | 18–19 / 52–54 s vs 20 / **27–31 s** |

Strata's default thinking limit (3/4 of max_tokens) made no difference: 201.5 and 86.9 tok/s with it off.

DeepSeek-V4.1-Flash on Maya's own quants against Project Maya v1.1.73:

| PC, file | tandem.cpp | Maya, no drafting | Maya + DSpark | first token 5.5k / 15.7k (ours vs Maya) |
|---|---|---|---|---|
| RTX 5090 + 128 GB, Maya-XS (fits in RAM) | **60.7** | 39.6 | 56.4 | **4.5 / 8.5 s** vs 7.7 / 9.5–9.7 s |
| RTX 5090 + 128 GB, Maya-S (178 GB, bigger than RAM) | **29.0** | 22.9 | 22.4 | 11.0 / 20.5 s vs 12.1–12.6 / **13.8 s** |
| RTX 5070 Ti + 64 GB, Maya-S (2.8× RAM) | **9.4** | 6.0 | 6.1 | 74 / 188 s vs **35–68 / 63–97 s** |

**Engine-level KL**: the same reference tokens (official FP8 weights,
8,687 positions) through Strata's engine with its serving settings: KL 0.0260, NLL 0.2067, top-1 94.80%; tandem.cpp on the same
IQ3_S file 0.0259 / 0.207 / 94.8%.

## Earlier tables (v0.1–v0.4 builds)

Single user, the first 200 GSM8K test questions (zero-shot chat, max 2048 tokens), decode tok/s. Every row gives both engines the same GPU memory and the same draft model, if any. llama.cpp is `f5222f25c` with its default placement (`-fit on`: dense layers on the GPU, then as many layers' experts as fit), with `-fitt` set to tandem.cpp's margin when a draft model is loaded. Each cell is "greedy / sampled", sampled being the model's recommended sampling (Qwen: T 1.0, top-k 20, top-p 0.95; gpt-oss: T 1.0, top-p 1.0; GLM: the server's defaults; DeepSeek: T 1.0, top-p 0.95). Accuracy is within run-to-run noise between the engines in every row. The scripts are in [`bench/`](../bench/).

Models: Qwen3.8-Flash-Next [ISTA GSQ-RCO](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF) IQ3_S (78 GiB) with the Q4_K MTP head above; gpt-oss-120b MXFP4 (`ggml-org`); Qwen3.6-35B-A3B UD-Q4_K_XL and GLM-4.5-Air Q4_K_M (unsloth); DeepSeek-V4.1-Flash-1017 [ISTA GSQ-RCO](https://huggingface.co/ISTA-DASLab/DeepSeek-V4.1-Flash-1017-GSQ-RCO-GGUF) IQ2_XS (139 GiB of routed experts). We tuned for Qwen3.8 only.

### Gaming PC: i5-14600KF (6P + 8E cores), DDR4-3200 (2 channels), Samsung PM9A1 (PCIe 4.0 NVMe)

DeepSeek-V4.1-Flash-1017 IQ2_XS, no draft, greedy, the first 40 GSM8K questions (this build, 2026-11-11). At 64 GB the model is
about twice the RAM, so most experts are read from the SSD; at 128 GB it fits.

| GPU, RAM | llama.cpp | tandem.cpp | | Strata v0.2.9 exact | time to first token (llama.cpp / tandem.cpp / Strata) |
|---|---|---|---|---|---|
| RTX 5070 Ti 16 GB, 64 GB | 7.4 | 14.2 / 13.3 | 1.9× | 10.4 | 14.4 / 6.3 / 6.8 s |
| RTX 5070 Ti + RTX 5060 Ti (32 GB, `-sm layer`), 64 GB | — | 16.9 | | — | — / 4.3 / — s |
| RTX 5090 32 GB, 128 GB | 11.5 | 33.1 / 32.2 | 2.9× | 28.0 | 3.4 / 1.7 / 4.0 s |

All cells answered 39–40 of 40 correctly. Two numbers for tandem.cpp are two runs (the ratio uses the first); the 32 GB row
splits the layers over both cards (the RTX 5060 Ti sits in an x4 slot).

### DeepSeek-V4.1 Maya-S and Maya-XS: the same weights on both engines

[Project Maya](https://github.com/mw00/project-maya) v1.1.64 (`d93d149`) is a MoE engine built for big models such as
DeepSeek-V4.1, with its own quants. Its Maya-XS v2 GGUF (routed experts IQ1_S/IQ2_XXS, 1.63 bpw, 117 GB; Engram tables Q6_K,
161 GB) runs on tandem.cpp after `tools/maya_ds41_convert.py` (it renames nothing in the model: it copies the three main
shards with their split count patched and joins the two Engram tables into the one tensor llama.cpp reads). Same files,
same PC (i5-14600KF, 128 GB DDR4-3200, RTX 5090, GPU 0 only), cold, greedy, the first 16 GSM8K questions twice, then two
prompts at 5.6k and 15.7k tokens (the same text for both):

| | decode tok/s | GSM8K | first token, 5.6k / 15.7k | decode after the long prompts |
|---|---|---|---|---|
| Maya, no drafting (as the tandem.cpp rows) | 39.0 / 39.6 | 16, 15 of 16 | 7.8 / 9.5 s | 26–43 |
| Maya with its DSpark drafter (its default here) | 54.9 | 15 of 16 | 7.9 / 9.7 s | 28–38 |
| tandem.cpp v0.3.0 | 60.3 / 59.4 | 15, 15 of 16 | 6.5 / 16.5 s | 63–68 |
| **tandem.cpp v0.4.0** | **60.0 / 60.9** | 15, 15 of 16 | **4.4 / 8.6 s** | **59–64** |

Maya's own installer and settings (reasoning effort high), DSpark off for the first row. The whole model fits in this PC's
VRAM + RAM, so neither engine reads the SSD after loading. Maya-S (routed experts IQ2_XXS/IQ3_XXS, the first and last four layers
IQ2_S/IQ3_S, 2.49 bpw, 178 GB; the same Engram tables) does not fit, so both engines read experts from the SSD. Same PC
and recipe, tandem.cpp v0.4.0:

| Maya-S | decode tok/s | GSM8K | first token, 5.6k / 15.7k | decode after the long prompts |
|---|---|---|---|---|
| Maya, no drafting (as the tandem.cpp rows) | 19.8 / 21.0 | 15, 15 of 16 | **12.5 / 14.0 s** | 7–11 |
| Maya with its DSpark drafter | 17.2 | 15 of 16 | 12.5 / 14.2 s | 9–13 |
| **tandem.cpp v0.4.0** | **25.4 / 25.2** | 15, 15 of 16 | 15–17 / 30–40 s | **29–34** |

Maya reads a long prompt faster here; the first long prompt is the one place tandem.cpp still loses when the experts do
not fit in RAM.

tandem.cpp runs the DeepSeek-V4.1 DSpark drafter too (`--spec-type draft-dspark -md dspark-DeepSeek-V4.1-Flash-1017-Q8_0.gguf`,
3 drafted tokens). It pays when the experts sit in VRAM (96 GB card: 132 vs 90 tok/s, [below](#server-epyc-9534-64-cores-ddr5-350-gbs-rtx-pro-6000-blackwell-96-gb-pcie-50-x16)), not on
this PC: Maya-XS 62.2 / 60.2 tok/s with it vs 60.3 / 60.7 without, IQ2_XS 28.1 vs 31.9. Checking four tokens at once touches
more experts, and the drafter takes 5 GB of the cache (hit rate 86% -> 74% on Maya-XS), so the extra expert traffic eats
what the drafted tokens save. Same PC, v0.4.0, drafter experts requantized to Q2_K.

To run it: download `peasantsmith/DeepSeek-V4.1-Flash-1017-Maya-GGUF` (folder
`Maya-XS/` or `Maya-S/`, MIT), then

```
python3 tools/maya_ds41_convert.py --maya-dir models/Maya-XS --out models/Maya-XS-llama --link
llama-server -m models/Maya-XS-llama/DeepSeek-V4.1-Flash-1017-Maya-XS-v2-IQ1_S-00001-of-00004.gguf \
  -ngl 99 -cmoe --moe-cache-mib auto -np 1 -c 32768
```

`--link` copies only the first shard (its metadata changes) and links the other main shards to Maya's files, so keep
those; it writes about 210 GB (the first shard and the two Engram tables joined), against a full copy without it (278 GB
for Maya-XS, 340 GB for Maya-S). If you run only tandem.cpp, Maya's two Engram shards can be deleted after the conversion.
Maya-S converts the same way (`--maya-dir models/Maya-S`; its first shard gets the Engram metadata names tandem.cpp reads).

### Consumer PC: i5-12400F (6 cores), 64 GB DDR4 (43 GB/s), RTX 4090 24 GB, PCIe 4.0 x16

The models are larger than RAM with the OS, so both engines read them from the mapped file.

| Model, setup | llama.cpp | tandem.cpp | | Strata exact |
|---|---|---|---|---|
| Qwen3.8 IQ3_S, MTP | 29.8 / 29.4 | 97.3 / 96.1 | 3.3× / 3.3× | 103.6 / 98.1 |
| gpt-oss-120b, no draft | 27.8 / 27.7 | 51.5 / 48.0 | 1.85× / 1.73× | |
| gpt-oss-120b, EAGLE3 | 25.2 / 24.2 | 41.1 / 42.0 | 1.63× / 1.74× | |
| DeepSeek-V4.1 IQ2_XS (3× RAM, from the SSD), no draft, earlier build (2026-11-01) | 6.86 / – | 5.99 / – | 0.87× | 8.41 / – |

- Strata exact is 6% ahead greedy and 2% sampled on Qwen3.8; its default mode, which skips low-ranked experts, gives 127.1. tandem.cpp's equivalent (`LLAMA_MOE_TAIL_SKIP=7`, approximate, off by default) gives 123.6 / 124.3.
- On 6 cores EAGLE3 slows both engines down: verifying several tokens multiplies the experts the CPU must run.
- DeepSeek-V4.1 is three times this PC's RAM. That row is from the earlier build, which was slower than llama.cpp when misses had to come from the SSD; the current build reads them ahead with several threads (see the gaming-PC rows). This PC's GPU has since changed, so the row was not re-measured.


The same PC with an RTX 5070 Ti 16 GB instead of the 4090, on an MSI M450 drive (DRAM-less, PCIe 4.0), models much larger than
RAM, from a cold start (page cache dropped), greedy, two runs per engine in mirrored order (2026-11-13).

Qwen3.8-Flash-Next IQ3_S + MTP (78 GiB) at 32 GB of RAM (`mem=` boot), 8K context, the first 16 GSM8K questions:

| | decode tok/s | read from the SSD | time to first token (median) |
|---|---|---|---|
| llama.cpp `-fit on`, the same MTP head | 14.9 / 15.1 | 294 / 290 GB | 5.1 / 5.3 s |
| tandem.cpp (defaults: store mode) | **34.3 / 35.0** | 231 / 232 GB | 2.9 / 4.1 s |
| Strata v0.2.9 exact, its own low-RAM setup and MTP settings | 30.3 / 32.0 | 492 / 418 GB | 4.3 / 4.2 s |

All cells 15 of 16 correct. Strata's draft settings are its own (not the same speculative decoding as the other two rows).

DeepSeek-V4.1-Flash IQ2_XS at 64 GB, no draft, the first 8 GSM8K questions: tandem.cpp
**10.4 / 10.2** tok/s (374 / 385 GB read, first token 13.4 / 14.4 s), Strata v0.2.9 exact 5.7 / 5.7 (746 / 708 GB, 15.0 / 13.7 s):
**1.8×**. All 8/8 correct. On the gaming PC's faster drive above the gap is smaller (1.4×).

### Older workstation: i9-10900X (10 cores), 8× DDR4-2666 (78 GB/s), RTX 4090 24 GB, PCIe 3.0 x16

| Model, setup | llama.cpp | tandem.cpp | |
|---|---|---|---|
| Qwen3.8 IQ3_S, MTP | 31.1 / 31.9 | 100.9 / 100.9 | 3.2× / 3.2× |
| gpt-oss-120b, no draft | 31.2 / 30.1 | 49.6 / 49.6 | 1.59× / 1.65× |
| gpt-oss-120b, EAGLE3 | 30.3 / 28.8 | 62.5 / 61.0 | 2.06× / 2.12× |

Drafts do not speed up llama.cpp on this host (Qwen3.8 29.8 without MTP): most of its experts run from system RAM, so verifying several tokens costs nearly as much as it saves.

### Workstation: Threadripper PRO 5955WX (16 cores), DDR4 (~80 GB/s read), RTX 6000 Ada 48 GB, PCIe 4.0 x16

| Model, setup | llama.cpp | tandem.cpp | | Strata exact |
|---|---|---|---|---|
| Qwen3.8 IQ3_S, MTP, whole card | 98.6 / 97.1 | 182.5 / 180.9 | 1.85× / 1.86× | 185.9 / 169.9 |
| Qwen3.8 IQ3_S, MTP, 12 GiB expert cache | | 131.1 / 131.9 | | 129.2 / 119.4 |
| Qwen3.8 IQ3_S, MTP, 4 GiB expert cache | | 91.3 / 89.5 | | 89.3 / 83.7 |
| gpt-oss-120b, whole card | 87.9 / 86.3 | 95.3 / 90.7 | 1.08× / 1.05× | |
| gpt-oss-120b, 24 GB budget | 47.1 / 47.3 | 70.7 / 68.5 | 1.50× / 1.45× | |
| gpt-oss-120b, EAGLE3, whole card | 88.7 / 84.4 | 136.7 / 126.8 | 1.54× / 1.50× | |
| Qwen3.6-35B-A3B (22.4 GB), fits | 173.1 / 173.1 | 179.2 / 179.2 | 1.04× / 1.04× | |
| Qwen3.6-35B-A3B, 16 GB budget | 117.8 / 117.8 | 148.6 / 148.6 | 1.26× / 1.26× | |
| Qwen3.6-35B-A3B, 12 GB budget | 97.2 / 96.5 | 136.5 / 135.4 | 1.40× / 1.40× | |
| Qwen3.6-35B-A3B, 8 GB budget | 79.8 / 80.0 | 115.0 / 114.5 | 1.44× / 1.43× | |
| GLM-4.5-Air (73 GB), whole card | 31.9 / 31.9 | 54.6 / 55.1 | 1.71× / 1.73× | |
| GLM-4.5-Air, 24 GB budget | 20.1 / 20.0 | 34.7 / 34.6 | 1.73× / 1.73× | |

The cache-size rows compare against Strata at the same expert cache; a "budget" is the GPU memory both engines may use. Strata's cache rows are its exact mode as above.

Long context: a 64k-token prompt of real text in a 72k context, two planted code words found in every cell, greedy. Prefill is the prompt's tokens over the time to its first token:

| Model, setup | prefill llama.cpp | prefill tandem.cpp | | decode llama.cpp | decode tandem.cpp | |
|---|---|---|---|---|---|---|
| Qwen3.8 IQ3_S, MTP | 600 tok/s (109 s) | 860 tok/s (76 s) | 1.44× | 88.0 | 158.4 | 1.80× |
| gpt-oss-120b, no draft | 660 tok/s (100 s) | 1150 tok/s (57 s) | 1.75× | 64.9 | 75.4 | 1.16× |
| gpt-oss-120b, EAGLE3 | 560 tok/s (117 s) | 1030 tok/s (63 s) | 1.84× | 85.0 | 125.5 | 1.48× |

On the GSM8K questions (short prompts) the time to first token is 1.6–2.7× lower with tandem.cpp in the rows we checked, except the consumer-PC DeepSeek row (22 s against 12 s, the SSD issue below).

### Server: EPYC 9534 (64 cores), DDR5 (~350 GB/s), RTX PRO 6000 Blackwell 96 GB, PCIe 5.0 x16

DeepSeek-V4.1-Flash-1017 IQ2_XS (139 GiB of experts), whole card, reasoning on:

| Setup | llama.cpp | tandem.cpp | | Strata v0.2.9 |
|---|---|---|---|---|
| no draft | 51.7 / 46.0 | 89.6 / 81.0 | 1.73× / 1.76× | 84.4 / 84.0 (6 experts, no DSpark) |
| DSpark draft (3 tokens) | 63.0 / – | 132.4 / 107.2 | 2.10× | 148.9 / – (its default: DSpark, up to 5 tokens) |

tandem.cpp caches 87 GiB of the experts here (98.7% hit rate). Five draft tokens with a 0.65 confidence cut-off, Strata's default, was slower for tandem.cpp (122.3).

### Quality

Qwen3.8 Q2_0, teacher-forced KL divergence of the next-token distribution against plain llama.cpp `f5222f25c` with the whole model on the GPU, 8-token batches:

| Build | mean KLD | same top token | PPL ratio |
|---|---|---|---|
| tandem.cpp, `LLAMA_MOE_NO_DEFAULTS=1` | 0.000000 | 100% | 1.000 |
| defaults, whole model on the GPU | 0.0091 | 96.1% | 0.999 ± 0.003 |
| defaults, 4 GiB cache + CPU path | 0.0093 | 95.5% | 1.000 ± 0.003 |

With the defaults off the patch computes exactly what llama.cpp does; the defaults change only rounding (fused kernels, kernel choice), and the cache and CPU path add ≤ 0.0002.


### AMD (ROCm), 2026-11-15

Tested on one PC: i7-8700 (6 cores), 64 GB DDR4, RX 7900 XTX 24 GB on PCIe 3.0, Ubuntu 26.04 with its ROCm 7.1 packages.
Qwen3.8-Flash-Next + MTP, greedy, the first 40 GSM8K questions, same day (2026-11-15), decode tok/s:

| Engine, quant | tok/s |
|---|---|
| tandem.cpp IQ3_S | 68.6 |
| Strata v0.2.9 IQ3_S | 69.7 |
| Strata's RDNA3 fork IQ3_S | 80.1 |
| Avalon IQ3_KS (its own quant) | 83.6 |

Stock Strata and tandem.cpp are level; on this 6-core CPU the AMD-specific engines are ahead. Most of that gap is the CPU's work on missed experts: IQ3_S is costly to
decode on a CPU, and Avalon's own quant types decode cheaply.

