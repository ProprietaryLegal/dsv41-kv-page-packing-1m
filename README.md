# DeepSeek-V4.1-Flash KV page packing and 1M context on DGX Spark / GB10 with x86 RTX 3090 offload

An experimental overlay for the HeNryous serving image combines KV page packing, head-node memory switches, and a longer context limit. It targets two DGX Spark (GB10) nodes plus an x86 offload host with RTX 3090s. GB10 validation in progress.

**Hugging Face:** [Card and files](https://huggingface.co/ProprietaryLegal/dsv41-kv-page-packing-1m) · **Stack overview:** [GitHub](https://github.com/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) · **Collection:** [All entries](https://huggingface.co/collections/ProprietaryLegal/deepseek-v41-flash-on-2x-dgx-spark-x86-offload-6adab604a517b426ded3916e)

<!-- pli:blurb:start -->

> Published by [PLI Labs](https://proprietarylegal.ai/labs.html), the open research arm of [Proprietary Legal Intelligence](https://proprietarylegal.ai). PLI builds AI systems around a legal organization's own work product, combining retrieval, staged workflows, validation, and attorney review. Visit [proprietarylegal.ai](https://proprietarylegal.ai) to explore our work and discuss what your organization needs.

<!-- pli:blurb:end -->

## Status

2026-10-23 — KV page packing validated on GB10 at 327,680 (production promotion pending); 524,288 serves correctly but is held for a prefill regression under investigation.

1,048,576 (1M) was not tested on GB10 in this window.

## What it is

The overlay applies HeNryous's patch chain to the Python sources of image `ghcr.io/henryous/deepseek-v41-1017-dgx-spark:v5`. The selected patches are `0051`, `0053`, `0057`, `0059` (code dependency only), `0060`, and `0076`. [The manifest](code/kvpack/MANIFEST.json) pins the donor revision, patch hashes, image, and base and overlay file hashes.

Packing places more compressed KV pages inside an existing allocator stride. It preserves the FP8 record format and the allocated pool size. The memory switches separately remove an unused residual buffer and change prompt-scratch lifetimes and aliasing. A context-limit change is another independent choice.

Mounted with no switch, the overlay runs the base image's code paths. The supplied experiment presets explicitly enable their named features and must be treated as opt-ins.

## Why

The base allocator leaves unused space in some compressed-KV groups. Packing increases the usable context pool without increasing its allocation. Separately, the longer context limit consumes head-node memory for position caches and workspaces. The memory switches are candidates for recovering some of that headroom, subject to an actual memory trace.

## When to use it

Use it in a supervised test window when a matching base image and remote-drafter capacity configuration already work. It is useful for testing whether more context fits, whether prefix-cache hits remain correct after packing, and whether the longer context limit stays within the available memory.

Complete the shorter-context inertness, packing, corruption, prefix-identity, memory, and forced-pressure checks before combining this overlay with prefix storage or scheduling changes. Validate combinations separately, even where their mounted files do not overlap.

The validated 327,680 configuration is a candidate for more KV capacity, with production promotion pending. Keep 524,288 in supervised testing while its cold-prefill regression is investigated; 1M remains untested on GB10.

## When not to use it

Do not use it as a qualified production upgrade or as evidence of improved decode speed. Allocator arithmetic does not establish usable request concurrency. Avoid an unreviewed image revision, mismatched switches across ranks, or a longer context limit that the remote drafter cannot serve.

Do not promote 524,288 while its prefill hold remains: cold single-stream prefill was about +35 % at the measured prompt lengths, including shorter prompts. Do not treat the repository's 1M name as a tested GB10 context limit.

Keep NVFP4 KV disabled. It is present only because the upstream patch chain depends on its code. Keep the optional KV-display switch disabled because its display module is not shipped.

## Requirements

- A working two-node DGX Spark (GB10) serve based on the exact image and source hashes in the manifest, plus the x86 RTX 3090 offload stack used by the experiment presets.
- An unpacked Python source tree of that base image, and a clean donor checkout at the manifest's pinned revision. The base image, model, and remote worker are external requirements.
- Bash, Python, Git, GNU patch, coreutils, and Docker supporting `--pull never`; staging also needs the existing node-access setup. The small CPU arithmetic and scorer checks use only the Python standard library.
- A launcher that mounts the same reviewed overlay and passes the same runtime variables to both containers, including the engine and worker processes.
- A remote drafter that accepts positions throughout the requested context range. Changing the target limit alone is insufficient.
- For the supervised long-context tests, keep **MemAvailable at or above 1.5 GiB on each node throughout**. This is a test stop condition, not a general reserve for other overlays.

## Measured effect

Measured 2026-10-23 on two DGX Spark (GB10) nodes, TP2, serving the [DeepSeek-V4.1-Flash-1017 EXL3 2.75 bpw legal-weighted build](https://huggingface.co/ProprietaryLegal/DeepSeek-V4.1-Flash-1017-EXL3-2.75bpw-LegalWeighted-DGX-Spark), with x86 RTX 3090 remote drafting. For `max-model-len=327680`, compare packing on with the same overlay and every switch off. Timing rows are single-stream except the explicitly concurrent row; memory attribution uses separate probed starts. Repeat counts are not included in this released summary.

| Metric / workload | Packing on versus every switch off |
|---|---|
| GPU KV cache | 2,502,600 versus 1,604,926 tokens (+55.9 %) |
| Capacity divided by full request length | 7.64× versus 4.90×; capacity ratios, not measured request concurrency |
| Allocator arithmetic | Predicted both token counts exactly |
| Memory saving, idle allocated memory | +1,339 MiB per rank (160 + 1,014 + 165), confirmed by the overlay memory probe |
| Lowest MemAvailable across these runs | 4.6 GiB |
| Step p50, short | +0.47 % |
| Step p50, 32K | −1.65 % |
| Step p50, 32K × 3 streams | +0.97 % |
| Step p50, 150K | −2.04 % |
| Cold-prefill TTFT, 32K | −0.17 % |
| Cold-prefill TTFT, 150K | −0.14 % |
| Acceptance | Unchanged |

Measured on the same date, hardware and served model at `max-model-len=524288`, with packing on. Comparisons use the packed 327,680 configuration; cold-prefill and 500K timing rows are single-stream. Repeat counts are not included in this released summary.

| Metric / workload | Result at 524,288 |
|---|---|
| GPU KV cache | 2,820,756 tokens (5.38× full request length; a capacity ratio) |
| Decode | Unchanged versus the packed 327,680 configuration |
| Cold single-stream prefill, measured prompt lengths | About +35 %; held for investigation |
| Cold-prefill TTFT, 150K | 93 s versus 69 s |
| Cold-prefill TTFT, 32K | 20 s versus 15 s |
| 500K single request | TTFT 322 s; decode 48 tok/s |

Correctness checks:

- At 327,680, the needle/cross-request corruption set passed 16/16 with no foreign codes.
- Prefix-cache identity checks passed at 327,680, with exact cached-token counts: `floor((n−1)/384)·384`, where `n` is the prompt-token count. Output comparisons use the engine's own run-to-run floor.
- Greedy and logit A/B checks at 327,680 were within the engine's own run-to-run floor.
- Forced pool pressure at 1.5 GiB/rank exercised preemption and eviction; all needles were found.
- At 524,288, needles at 450K and 500K, prefix identity, and the corruption set all passed.

The 524,288 prefill regression is under investigation. A prefill path whose cost may scale with `max-model-len` is a hypothesis; the cause is not proven. 1,048,576 (1M) was not tested on GB10 in this window.

## Upstream claims

**HeNryous authored the upstream patches.** They claim to enable KV page packing and head-node memory switches that let a longer context limit fit. See the [upstream repository](https://github.com/HeNryous/deepseek-v41-1017-dgx-spark) and its [patch directory](https://github.com/HeNryous/deepseek-v41-1017-dgx-spark/tree/main/patches).

The packing switch places compressed pages inside the existing allocator stride. The head-memory switches remove an unused residual buffer, make prompt scratch transient, allow expert scratch to alias that prompt scratch, and adjust the indexer prefill buffer allocation. The alias switch requires transient allocation.

Transient prompt scratch returns to torch's caching allocator, rather than directly to the system. Its scratch exists after the startup profile, rather than appearing only after a long request. Lower torch-allocated memory does not imply an equal increase in system MemAvailable. Combined switches do not establish the isolated effect of transient scratch on this setup.

## Deploy

This is a source overlay for integration with an existing deployment, not a complete serving launcher. The commands below are for an operator's supervised window. Set `NODE_A` and `NODE_B` to the two nodes using the existing access configuration.

- Obtain the exact base-image source and clean donor checkout identified by the manifest. From this repository root, verify the pinned rebuild and the shipped overlay:

   ```bash
   SRC=/data/image-source DONOR=/data/henryous-pinned \
     bash code/kvpack/build_overlay.sh
   bash code/kvpack/stage_overlay.sh verify-checkout
   ```

   The rebuild verifies patch hashes and checks rebuilt files and manifest against the shipped copy. A mismatch stops integration; obtain matching sources before continuing.

- Check the installed image on each node, then stage a versioned, hash-verified overlay:

   ```bash
   bash code/kvpack/stage_overlay.sh check "$NODE_A" "$NODE_B"
   bash code/kvpack/stage_overlay.sh stage "$NODE_A" "$NODE_B"
   bash code/kvpack/stage_overlay.sh args
   ```

   The image check does not pull a missing image. Add the emitted read-only bind mounts to both containers in the existing launcher. Store the literal mount arguments with the launcher configuration; avoid evaluating a staging helper every time a configuration file is sourced. Preserve the emitted node-local path expansion.

- Start with every added switch unset and the base context limit. Use `verify-running --expect ""` with both node arguments to verify the actual mounted bytes and absence of added runtime overrides. Compare allocator lines, greedy output, and prefix identity against the same worker and serving configuration without the overlay. Stop if the overlay is not inert.

- Only then opt into the candidate runtime settings on both ranks:

   ```text
   VLLM_DSV41_KV_PAIR=3
   VLLM_DSV41_DROP_MTP_BUFFER=1
   VLLM_EXL3_TF_PROMPT_TRANSIENT=1
   VLLM_EXL3_TF_PROMPT_ALIAS=1
   VLLM_INDEXER_PREFILL_BUFFER_FACTOR=4
   ```

   The alias switch requires transient allocation. Run `verify-running --expect` with the complete intended switch set after every start. Keep timed runs free of the memory probe; use separate probed starts for memory attribution. The experiment presets use the async process-proxy base so the control and candidate share the same worker path.

- Test packing at the shorter context limit before testing **`max-model-len=1048576`**. Exercise cold/hit/fresh-salt output identity, partial-prefix hits, cross-request corruption, preemption, eviction, and longer-context recall. A fresh-salt control must report no cached tokens. Hold the worker and serving base fixed between comparisons. Stop on incorrect output, an unexpected cache count, drafter failures at longer positions, or the memory floor. Restore the base serve on any failure.

The original-layout presets are integration references. They depend on files absent from this standalone package; see the limits below.

## Rollback

Remove the overlay bind mounts and its runtime-variable overrides from both node configurations. Restore the previous context limit and launcher configuration, then stop and restart the existing serve using its normal scripts. Verify that both containers use the base files, allocator lines, and switch state before resuming traffic. The staged files can remain on disk; without the mounts they are inactive.

## Limits and known issues

- **Production promotion is pending at 327,680 despite successful GB10 validation.** These bounded checks do not qualify other images, workloads, or overlay combinations.
- **524,288 is held for a prefill regression under investigation.** Correctness passed and decode was unchanged, but cold single-stream prefill was about +35 % at the measured prompt lengths: 150K TTFT 93 s versus 69 s; 32K 20 s versus 15 s. The cause is not proven.
- **1,048,576 (1M) was not tested on GB10 in this window.** No result at 524,288 qualifies the larger limit.
- Packing changes prefix-hit granularity while hashes retain their existing granularity. A hit can need more prompt work. The optional EAGLE trailing-block-drop override has no effect with bounded sliding-window replay in the reviewed serve and stays off.
- The ungated state-bucket split is inert for this model because its compressor-ring caches have one page size. That conclusion does not rely on the KV record format.
- `test_stage_overlay.sh` assumes the original repository layout, root deployment settings, the full generated config set and its bases, and an unpacked base-image source tree. These are not all included. `make_configs.py --check` also resolves paths through that original layout. Neither is a self-contained standalone suite here.
- `stage_overlay.sh` contains original-layout paths in its experiment-overlay output. Its image check needs the matching image on the nodes; its offline tests need the unpacked image source. The direct `verify-checkout`, `check`, `stage`, and mount-argument modes should be distinguished from the original-layout `overlays` mode.
- `build_overlay.sh` needs the external base sources and pinned donor, even though it can be invoked from this package. The overlay Python files import the rest of the image's vLLM and plugin tree; this repository is not an installable replacement for those packages.
- The memory probe creates no parent directory. The original presets require a flat prefix inside the container's existing writable cache directory. Use its layer selection for phase records and keep it out of timed runs.
- The pressure-test setup must force preemption even with packing enabled. Its model uses the sliding window actually held during chunked prefill, rather than its admission cap; it predicts whether preemption is required, not its count.
- Prefix storage and scheduling changes need their own memory budgets and combined-runtime tests. A successful isolated overlay test would not qualify those combinations.
- The benchmark helpers require a user-supplied public corpus for prompt construction. No corpus or benchmark captures are included.

## Repository layout

Paths are relative to the standalone repository root:

| Path | Purpose |
|---|---|
| [code/kvpack/MANIFEST.json](code/kvpack/MANIFEST.json), [FILES](code/kvpack/FILES) | Pinned provenance and per-file hashes; overlay path inventory |
| [code/kvpack/build_overlay.sh](code/kvpack/build_overlay.sh) | Rebuild and compare against the pinned external image source and donor |
| [code/kvpack/stage_overlay.sh](code/kvpack/stage_overlay.sh), [test_stage_overlay.sh](code/kvpack/test_stage_overlay.sh) | Hash-checked staging and live verification; original-layout offline regression suite |
| [code/kvpack/kv_arith.py](code/kvpack/kv_arith.py) | Standalone capacity and pressure arithmetic with validation anchors |
| [code/kvpack/make_configs.py](code/kvpack/make_configs.py) | Original-layout preset generator and drift check |
| [code/kvpack/overlay/vllm/models/deepseek_v41](code/kvpack/overlay/vllm/models/deepseek_v41) | Attention specs, compression/cache operations, sparse attention, drafter integration, and model-buffer switches |
| [code/kvpack/overlay/vllm/v1/core/kv_cache_utils.py](code/kvpack/overlay/vllm/v1/core/kv_cache_utils.py), [kv_cache_interface.py](code/kvpack/overlay/vllm/v1/kv_cache_interface.py) | Allocator grouping, capacity, and packed-cache specifications |
| [code/kvpack/overlay/vllm/v1/worker](code/kvpack/overlay/vllm/v1/worker) | Worker cache views, model runner, and optional memory probe |
| [code/kvpack/overlay/vllm_exl3_moe](code/kvpack/overlay/vllm_exl3_moe) | Prompt scratch lifetime and TensorFold expert-scratch aliasing |
| [code/kvpack/needle.py](code/kvpack/needle.py), [prefix_identity.py](code/kvpack/prefix_identity.py) | Recall, cross-request corruption, and prefix output-identity helpers; included because the hardware test plan uses them |
| [code/launch/configs](code/launch/configs) | Async-base presets for packing, longer context, and the unpacked longer-context control |
| [NOTICE](NOTICE), [LICENSES](LICENSES) | Third-party attribution, retained notices, and license texts |

The documented home for `needle.py` and `prefix_identity.py` is [dsv41-longctx-bench-tools on GitHub](https://github.com/ProprietaryLegal/dsv41-longctx-bench-tools), with its [HF card](https://huggingface.co/ProprietaryLegal/dsv41-longctx-bench-tools).

## Tests

From the published repository root, run the self-contained CPU checks:

```bash
python3 -B code/kvpack/kv_arith.py
python3 -B -c 'import runpy; runpy.run_path("code/kvpack/prefix_identity.py")["self_test"]()'
```

The first checks capacity anchors and pressure verdicts; the second checks the identity scorer, expected-hit calculation, comparison floor, and fresh-salt-control guard. Neither connects to a model server. To exercise prompt construction as well, supply a public corpus in the helper's expected format:

```bash
python3 -B code/kvpack/prefix_identity.py --dry-run --n 8000 --data-dir /data/public-corpus
python3 -B code/kvpack/needle.py --dry-run --ctx 8000 --data-dir /data/public-corpus
```

The corpus needs `opinions_cl.jsonl` and `filings_recap.jsonl`; dry-run token counts are estimates. These checks are not model recall or hardware correctness tests.

CPU checks passed after the review fixes. To reproduce the original-layout regression suite, first reconstruct the original `offload/kvpack` layout with its root deployment settings, both base configs, all generated configs, pinned donor, and unpacked source tree. Then run from that deployment root:

```bash
SRC=/data/image-source DONOR=/data/henryous-pinned \
  bash offload/kvpack/test_stage_overlay.sh
```

The original-layout CPU checks do not establish a standalone staging-suite pass for this package.

## Credits and licences

**HeNryous authored the page-packing and head-memory patches.** This release packages their selected changes with reproducibility, staging, arithmetic, configuration, and validation helpers. The image builds on sudoingX's serving recipe and vLLM. TensorFold supplies the adapted expert wrapper; turboderp's exllamav3 supplies the runtime EXL3 implementation and format used by the base stack.

HeNryous's patches, the vLLM sources, and the image plugin are Apache-2.0. The adapted TensorFold source is Apache-2.0 with its retained notices, including the legacy MIT notice. The external exllamav3 dependency is MIT. [NOTICE](NOTICE) maps these components to the retained texts in [LICENSES](LICENSES), including the complete upstream notices. Model licensing remains separate; see the [public base model](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017). No model weights are included. License sources: [HeNryous](https://github.com/HeNryous/deepseek-v41-1017-dgx-spark/blob/main/LICENSE), [vLLM](https://github.com/vllm-project/vllm/blob/main/LICENSE), and [TensorFold](https://github.com/bertholomus/TensorFold/blob/deepseek-v41-tp2/LICENSE).

## Related

| Project | GitHub | Hugging Face |
|---|---|---|
| Stack overview | [Repository](https://github.com/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) | [Card](https://huggingface.co/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) |
| Long-context benchmark tools | [Repository](https://github.com/ProprietaryLegal/dsv41-longctx-bench-tools) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-longctx-bench-tools) |
| Remote RTX 3090 drafter | [Repository](https://github.com/ProprietaryLegal/dsv41-remote-dspark-drafter-3090) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-remote-dspark-drafter-3090) |
| Async handoff | [Repository](https://github.com/ProprietaryLegal/dsv41-async-hostflag-handoff) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-async-hostflag-handoff) |
| Two-card drafter | [Repository](https://github.com/ProprietaryLegal/dsv41-dspark-drafter-tp2-nvlink) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-dspark-drafter-tp2-nvlink) |
| Remote Engram | [Repository](https://github.com/ProprietaryLegal/dsv41-remote-engram-rdma) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-remote-engram-rdma) |
| RDMA transport | [Repository](https://github.com/ProprietaryLegal/ofl-rdma-transport) | [Card](https://huggingface.co/ProprietaryLegal/ofl-rdma-transport) |
| Native drafter | [Repository](https://github.com/ProprietaryLegal/dsv41-native-1017-drafter-sm86) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-native-1017-drafter-sm86) |
| Exact acceptance ceiling | [Repository](https://github.com/ProprietaryLegal/dsv41-exact-acceptance-ceiling) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-exact-acceptance-ceiling) |
| Findings | [Repository](https://github.com/ProprietaryLegal/dsv41-spark-offload-findings) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-spark-offload-findings) |

Also see the [HeNryous image fork](https://github.com/ProprietaryLegal/deepseek-v41-1017-dgx-spark), [sudoingX recipe fork](https://github.com/ProprietaryLegal/dgx-spark-deepseek-v41), [research hub](https://github.com/ProprietaryLegal/pli-labs/tree/main/research/dgx-spark-offload), and [HF collection](https://huggingface.co/collections/ProprietaryLegal/deepseek-v41-flash-on-2x-dgx-spark-x86-offload-6adab604a517b426ded3916e).

<!-- pli:footer:start -->

---

## About PLI Labs

“Your proprietary work product is your proprietary intelligence.” PLI Labs is the open research arm of Proprietary Legal Intelligence. We publish legal model prunes, serving recipes for owned hardware, and open-source tools so readers can inspect our methods and test our claims before hiring us.

Founded by a practicing trial lawyer, PLI builds AI systems around a legal organization's own work product: retrieval, staged workflows, validation, and attorney review. The approach grew from systems running on live matters in a litigation practice. Our consulting work supports deployment through approved frontier APIs, private cloud, or open-weight models on hardware you own. Explore the research or contact us about your organization's work.

[Website](https://proprietarylegal.ai) · [PLI Labs](https://proprietarylegal.ai/labs.html) · [Research](https://proprietarylegal.ai/research/index.html) · [GitHub](https://github.com/ProprietaryLegal) · [Hugging Face](https://huggingface.co/ProprietaryLegal) · [Contact](https://proprietarylegal.ai/contact.html)

Proprietary Legal Intelligence LLC is a technology consultancy, not a law firm, and does not provide legal services or legal advice.

<!-- pli:footer:end -->
