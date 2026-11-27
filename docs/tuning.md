# Tuning tandem.cpp

For people who want to know what the flags do and which knobs exist. To just run a model, the [README](../README.md)'s
quick start and model commands are enough: the defaults below are what we measured to be best on the PCs we have.

## How it works and the flags

How it works: the experts live in host RAM, and the GPU memory left over holds a cache of recently used experts. The experts a token needs that are not in the cache are split between the CPU, which computes its share while the GPU runs the cached ones, and an upload into the cache. The split follows the measured CPU and PCIe speeds, using Strata's rule; while it runs, tandem.cpp tunes the split around the rule and saves the result for the next start (`LLAMA_MOE_SHARE_TUNE`, below).

Any MoE model whose experts do not fit in VRAM:

```
llama-server -m model.gguf -ngl 99 -cmoe --moe-cache-mib auto -lm none -np 1 -c 16384
```

- `-cmoe` keeps every expert in host RAM. `--moe-cache-mib auto` gives the GPU memory left after the model, the KV cache and the compute buffers to the expert cache, minus `--moe-cache-margin-mib` (default 512 MiB; a draft model's GPU size is added automatically). A fixed size in MiB also works.
- `-lm none` reads the weights into RAM instead of mmapping the file.
- Warm start: `LLAMA_MOE_HOT_SAVE=hot.txt` saves the experts a session used when the server exits; `LLAMA_MOE_HOT_LIST=hot.txt` loads them into VRAM at the next start (both can point to the same file).
- `-np 1 -c 16384`: one conversation with a 16k-token context. Without them llama-server opens 4 slots that share a small context, too short for a thinking model. `-c` is the total context; the KV cache comes out of the GPU memory the expert cache would get, so set it to what you need (on Qwen3.8, about 0.7 GiB per extra 16k tokens).
- The startup line `failed to fit params ... n_gpu_layers already set by user to 99, abort` is expected: `-ngl 99` turns llama.cpp's automatic placement off, and the expert cache does the placement instead.
- **Less RAM than the model** (e.g. Qwen3.8 IQ3_S, 78 GiB, on a 64 GB PC): leave out `-lm none`. The file stays mapped; the experts are copied once into pinned memory when the RAM holds them plus 10 GiB, and the rest is read from the file.
- **Much less RAM (e.g. a 16 GB PC):** also pass `--cache-ram 0`. llama-server's prompt cache otherwise takes up to 8 GiB of RAM and pushes the experts out of the page cache.
- Supported expert types for the CPU path: SwiGLU experts (fused or separate gate/up, optional clamp) and gpt-oss's SwiGLU-OAI with biases. For other types the server logs a warning and uploads every miss instead (correct, slower).


## The launcher (`tandem-run.py`) in detail

### Reading the plan

```
== Hardware ==   GPU, VRAM, PCIe, physical cores, RAM (the lower of MemTotal and the container limit), free disk, kernel
== Model ==      which file, which draft
== Plan ==       <flag>   <why>          one line per choice; "(no -lm none)" means a flag was left out on purpose
                 note: / WARNING:        things to know about this machine (below)
== Calibration == cached, or "will run on first start"
== Command ==    copy this to run it by hand
```

### Dry run

`--dry-run` prints the plan and starts nothing, downloads nothing. Use it to see what the launcher would do, or to test other
hardware (`--assume-vram-gib`, `--assume-ram-gib`, `--assume-cores` apply the rules for a smaller machine; they do not limit the
real hardware).

### Calibration

The first start for a machine, model and server build loads the model once per candidate draft count (the server cannot change it
per request), measures decode speed on three short prompts, keeps the fastest and saves it to
`~/.cache/tandem/<key>.json` (`$TANDEM_CACHE_DIR` moves it). It adds about 1-2 minutes after the first model load. Later starts reuse it.
`--calibrate` redoes it and also tries one fewer draft token; `--no-calibrate` skips it and uses the rules.

### Notes the plan may print

- **The model is larger than RAM**: the experts are read from the SSD, so speed depends on the SSD, and the first requests are slower
  while the cache warms. Below 0.6x the model size in RAM the drive sets the speed (DeepSeek-V4.1 on 64 GB: ~12 tok/s from a
  PCIe 4.0 drive, ~8 from a DRAM-less one).
- **Older kernels (before 6.1)**: they reclaim mapped model pages poorly. Expect slow loads when RAM is below the model size + 8 GiB.
- **"This build predates the low-RAM fixes"**: your tandem.cpp is older than the RAM fixes. Update it; until then the launcher sets
  `LLAMA_MOE_PIN_EXPS=0` and `LLAMA_PLE_GPU_COPY=0` when RAM is limited (a container limit is not seen by old builds).
- In a container the launcher uses the memory limit (`memory.max`), not the host's RAM.

### Other options

`--port`, `--ctx` (default 16384), `--model-path` / `--draft` (your own files, e.g. another quant), `--no-draft`, `--gpu N`,
`--extra "<llama-server flags>"` (appended last).
`--fast` adds `LLAMA_MOE_TAIL_SKIP=7`: faster but **approximate** (quality: TODO).


## Defaults (environment variables)

The server applies these when the variable is unset (logged at startup); `VAR=0` turns one off, `LLAMA_MOE_NO_DEFAULTS=1` turns all off. All are exact or differ only in floating-point rounding (kernel choice); the Qwen3.8-specific ones act only on that architecture.

| Variable | Default | What |
|---|---|---|
| `LLAMA_MOE_CPU` | `auto` | CPU/GPU split for missed experts (`0..1` = fixed share uploaded) |
| `LLAMA_MOE_PREFILL_CPU` | `auto` | the split for prompt batches |
| `LLAMA_MOE_SHARE_TUNE` | `1` | with `LLAMA_MOE_CPU=auto`, decode tries the uploaded share at the rule's ±0.1 in short blocks, moves 0.1 when one side is clearly faster over several trials (v0.5.2), and saves the offset per model, GPUs and cache size, RAM and threads (v0.5.1) |
| `LLAMA_MOE_TUNE_FILE` | `$XDG_CACHE_HOME/tandem/share-tune.tsv` (else `~/.cache/…`) | where the tuner keeps its offsets; `0`: no file (each start tunes from the rule) |
| `LLAMA_MOE_CPU_THREADS` | cores − 1 | CPU threads for missed experts |
| `LLAMA_MOE_CPU_RB`, `LLAMA_MOE_CPU_DOWN_PE` | `64`, `64` | CPU expert job work-item layout |
| `LLAMA_MOE_STORE_AUTO` | `1` | store mode (below) when `LLAMA_MOE_ARENA_MIB` is unset and either the experts outside VRAM exceed the RAM, or all experts exceed 85% of the RAM and VRAM + the block can hold every one |
| `LLAMA_MOE_ARENA_MIB` | unset | `auto` forces store mode: the experts outside VRAM live in one pinned block of RAM (the whole RAM less the reserve), read with direct I/O |
| `LLAMA_MOE_ARENA_RESERVE_MIB` | 2048 + RAM/64 | store mode: RAM left free (on top of the kernel's CMA region, which pinned memory cannot use) |
| `LLAMA_MOE_ARENA_FILL` | `1` | store mode: when VRAM + the block hold every expert, place them all at start |
| `LLAMA_MOE_ARENA_EXCL` | `1` | store mode: an expert is in VRAM or in the block, not both; the VRAM victim is copied back to RAM |
| `LLAMA_MOE_ARENA_ASYNC` | `1` | store mode: decode reads from the SSD overlap the compute |
| `LLAMA_MOE_ARENA_EARLY`, `LLAMA_MOE_ARENA_EARLY_EV` | `1`, `1` | store mode: the VRAM victims are copied back to RAM on a second stream while the GPU computes, ordered by GPU events (v0.5.0; Maya-S on 128 GB 25.2 → 29.0 tok/s) |
| `LLAMA_MOE_FILL_UPLOAD` | `0` | store mode with a complete fill: `1` uploads every miss (the v0.4.0 behaviour); by default the CPU keeps its measured share (Qwen3.8 IQ4_XS on 64 GB 25.0 → 44.6 tok/s) |
| `LLAMA_MOE_STREAM_READ` | auto | store mode whose RAM store holds at least 60% of the experts: long prompts still use layer streaming, reading each layer's missing experts from the SSD one layer ahead (`1`: always, `0`: never) |
| `LLAMA_MOE_PIN_EXPS` | auto | mmap mode: copy the mapped experts to pinned memory when the RAM holds them + 10 GiB |
| `LLAMA_MOE_UBATCH` | `4096` (`8192` with layers over 2 GiB) | with an expert cache and `-b`/`-ub` left at their defaults, prompts run in batches of this many tokens (`0`: `-ub`, 512). Draft contexts keep 512 |
| `LLAMA_MOE_STREAM` | `1` | prompt batches over 1024 tokens copy the next layer's experts into VRAM while the current layer computes (one GPU, experts in pinned RAM) |
| `LLAMA_MOE_OVERLAP` | `1` | where streaming does not fit, a prompt batch's expert copies overlap the compute of the previous expert range (experts in pinned RAM) |
| `LLAMA_MOE_FLEX` | `1` | one GPU with CUDA VMM: the larger prompt buffers borrow expert-cache VRAM while a prompt runs (`0`: the cache is sized for them) |
| `LLAMA_MOE_ARENA_FILL_PART` | `1` | store mode: fill the RAM store even when it is up to 3% short (the rest stays on the SSD) |
| `LLAMA_MOE_GATHER` | `0` | `1`: a prompt batch's expert copies in one GPU kernel (slower than the copy engines on the GPUs we tried) |
| `GGML_SCHED_BATCH_INPUTS` | `3` | a graph's host inputs in one packed upload |
| `GGML_CUDA_GRAPH_KEY_SHAPE` | `1` | one CUDA graph per batch shape |
| `GGML_CUDA_MMVF_MAX` | `8` | vector kernel for float weights up to 8 columns |
| `GGML_CUDA_Q8_CACHE` | `1` | reuse quantized activations across matvecs |
| `GGML_CUDA_CPY_RUN` | `1` | consecutive same-shape copies in one kernel |
| `LLAMA_MOE_RB_PACK` | `1` | the routing results read back to the CPU in one copy |
| `LLAMA_GRAPH_POOL` | `6` | keep allocated graphs for alternating small batch shapes; with `--moe-cache-mib auto` fewer (or none) at long contexts |
| `LLAMA_GRAPH_POOL_MIB` | `256` | VRAM those pooled graphs may take, measured at the full context and left out of the auto cache size |
| `LLAMA_SRV_USER_BREAK` | `0` | no prompt-cache checkpoint at every user message |
| `LLAMA_MTP_DEFER` | `1` | MTP catch-up decode deferred to the next draft |
| `LLAMA_HC_FUSE`, `GGML_CUDA_HC_MIX_FUSE` | `1` | hyper-connection fusions (Qwen3.8) |
| `LLAMA_GDN_DIRECT`, `LLAMA_GDN_GATES` | `1` | Qwen3.8 gated delta net |
| `LLAMA_GDN_REPLAY` | `1` | Qwen3.8 with speculative decoding, one sequence: one recurrent state plus the last step's inputs, replayed after a rollback, instead of a snapshot per draft position (450 → 135 MiB with 3 drafts) |
| `LLAMA_CONV_CONT`, `LLAMA_CONV_TAIL` | `1` | Qwen3.8 conv state |
| `LLAMA_PLE_GPU` | `1` | per-layer embeddings read by the GPU from pinned host memory |
| `LLAMA_PLE_GPU_COPY` | auto | Linux 6.8+ (file mmaps cannot be pinned): copy the table to pinned memory, only when the RAM holds the model + 8 GiB |

