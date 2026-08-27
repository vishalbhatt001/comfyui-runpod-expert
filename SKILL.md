---
name: comfyui-runpod-expert
description: "Everything learned deploying ComfyUI-based AI generation (video, music, audio) to RunPod GPU pods — exact model files/nodes for MiniMax H3, Wan 2.2, ACE-Step 1.5, Gemma 4, LTX 2.3; the Docker/dependency bugs that break these builds and their fixes; RunPod API deployment patterns and gotchas; cost-estimation method; Apple Silicon (MPS) limits. Load this before touching any new ComfyUI model integration or RunPod deployment — it's cheaper to read this than to rediscover a bug already fixed here."
---

# ComfyUI + RunPod Expert

A living record of hard-won, verified facts from building four production pipelines (`minimax-h3-studio`, `wan22-music-video`, `ace-music-studio`, `music-video-generator`) — all in `~/Documents/ComfyWorkspace/`, all public GitHub repos under `vishalbhatt001`. This skill exists so a *new* debugging session doesn't rediscover a bug already fixed here, and so a *new* model integration doesn't waste RunPod money guessing at node names that could've been verified in five minutes.

**Update discipline**: whenever a new bug is found and fixed, or a new model integration is verified, add it here — a new row in the relevant table, or a new subsection — before moving on. Then commit and push (see "Keeping this current" at the bottom). Don't let this go stale; it's only valuable if it stays accurate.

---

## 1. The one rule that prevents the most expensive mistakes

**Never write a ComfyUI API graph (`class_type`, input names, exact string values) from memory or from a blog post. Verify every node's exact schema by fetching ComfyUI's own source directly** (`raw.githubusercontent.com/comfyanonymous/ComfyUI/master/comfy_extras/<file>.py` or the equivalent for a custom node pack), or by fetching an official reference workflow JSON and reading its actual `class_type`/`widgets_values`.

Why this matters more than it sounds: a wrong node name or input key doesn't fail gracefully — it either gets silently rejected by ComfyUI (400 error, cheap) or, worse, gets *accepted* with wrong values and burns real GPU-minutes producing garbage before you notice (expensive). Every model integration below was built by fetching source first, and it caught real gotchas every single time (see §4).

Fast way to do this: launch a research agent/fork with WebFetch access, point it at the exact GitHub raw URL, and ask it to quote the actual class definition (inputs, outputs, defaults) rather than summarize. Don't trust a secondary blog's paraphrase of a node's parameters. If Comfy Org's own `comfy-cloud` plugin tools (`search_nodes`, `get_node`) are available in a session, they're a faster, equally authoritative alternative to manual source-fetching — see §8.

---

## 2. RunPod deployment — patterns and gotchas

### Two ways to deploy, and when each API surface has fields the other doesn't
- **GraphQL** (`https://api.runpod.io/graphql?api_key=...`): `podFindAndDeployOnDemand`, `podStop`, `podResume`, `podTerminate`, `saveTemplate`. Introspection is disabled on this account — you can't query the schema, only test mutations empirically and read error messages.
- **REST** (`https://rest.runpod.io/v1/...`): richer schemas for some things GraphQL doesn't expose well — notably `PodCreateInput.networkVolumeId` for attaching a persistent Network Volume. Fetch `https://rest.runpod.io/v1/openapi.json` (with your API key as a Bearer token) to get the exact request schema when GraphQL's lack of introspection leaves you guessing.
- Always send a `User-Agent` header on raw `curl`/`urllib` calls to RunPod's API — a default Python `urllib` request with no `User-Agent` gets a bare `403 Forbidden` that has nothing to do with auth.

### Deploying: pass everything explicit, don't trust `templateId` alone
Tested directly: calling `podFindAndDeployOnDemand` with only `templateId` set (relying on the saved template's `volumeInGb`/`containerDiskInGb`/`volumeMountPath`) did **not** reliably carry those settings through to the deployed pod — the resulting pod's actual disk sizes didn't match the template. Fix: pass `containerDiskInGb`, `volumeInGb`, `volumeMountPath`, `ports`, and `imageName` **explicitly in the deploy mutation itself**, every time, even when a template exists. Verify after deploying with a `pod(input:{podId})` query for `containerDiskInGb`/`volumeInGb`/`volumeMountPath` before trusting it.

### The volume-mount-path trap
If your Docker image bakes the app/ComfyUI under `/workspace` (a common pattern), **never** mount a persistent volume at bare `/workspace` — it overlays and hides everything the image put there, and the container fails to boot (its own entrypoint script "doesn't exist" from its own point of view). Scope the mount to a subpath that only needs to persist, e.g. `/workspace/ComfyUI/models` — this was the single most disruptive bug across this whole body of work, and it's entirely avoidable by never mounting at the exact path your Dockerfile's `WORKDIR`/copied code lives under.

### GPU capacity fluctuates fast on Community Cloud
`SUPPLY_CONSTRAINT` / "no instances available" is common even when RunPod's own UI shows "Low" (not zero) stock. Don't treat one failed deploy attempt as "unavailable" — retry every ~20s, alternating between GPU variants if more than one is acceptable (e.g. A100 PCIe and A100 SXM). It usually succeeds within a handful of attempts. A tight retry loop (bash `for`/`sleep 20`, quiet — don't log every failed attempt or you'll spam notifications) is the reliable pattern; a single attempt is not a reliable signal of true availability.

### Per-pod "Volume disk" vs. a real Network Volume — pick deliberately
- The **Volume disk** configured via `volumeInGb` on a pod/template is **ephemeral to that pod** — wiped on *terminate* (not on *stop*). Every fresh pod re-downloads its models.
- A separate, reusable **Network Volume** (`POST /v1/networkvolumes`, region-pinned) persists across pod terminations — but a pod using one is locked to that volume's specific datacenter, which compounds badly with GPU-availability fluctuation (you can't just retry in a different region). In practice, for a single/occasional-use case, re-downloading on a fresh per-pod disk was *less* friction than fighting one region's capacity. A Network Volume earns its cost when you're redeploying the same pod type frequently enough that re-download time actually dominates.
- Network Volumes bill for storage continuously even while unattached (~$0.001/GB-day observed) — delete one if you're not actively using it.

### RunPod's official ComfyUI template exists and is often the right call
Template IDs on this account: `cw3nka7d08` ("ComfyUI - CUDA 12.8"), `2lv7ev3wfp` ("ComfyUI - CUDA 13") — both expose port `8188/http` by default. If a model ships **natively** in ComfyUI (no custom quantization stack, no exotic compiler/torch-version requirement), there's no reason to build a custom Docker image — deploy from this template and just download the model weights on first boot. Reserve a custom image (see §3) for models that genuinely need a non-stock dependency stack (MiniMax H3's `comfy-kitchen`/triton stack did; Wan 2.2 and ACE-Step's stock fp16/bf16 nodes don't).

### SSH port vs. web terminal
Exposing TCP port 22 on a pod does **not** mean SSH works unless the image actually runs an sshd (most custom images don't, and won't unless you add one). For live debugging inside a running pod, use RunPod's own browser-based **web terminal**, not an SSH client pointed at the exposed port.

### Cost discipline
Billing is per-second while a pod *runs*, generating or not. Always confirm current status (`list`/`pod` query) and stop/terminate explicitly when done with a session — don't rely on remembering. When budget is tight, prefer building/testing logic locally (syntax checks, unit-testing parsers and workflow-graph builders against sample data) and defer live pod tests until ready to spend.

---

## 3. Docker image gotchas for a from-scratch ComfyUI build

Only build a custom image when the model needs a non-stock dependency stack. When you do, these are the traps hit building one for MiniMax H3 (int8/nvfp4 quantization via `comfy-kitchen` + triton):

| Symptom | Root cause | Fix |
|---|---|---|
| `OSError: Invalid cross-device link` moving a downloaded model file | Downloaded to one path, `os.replace()`'d across a mounted-volume boundary | Download **directly** to the file's final destination path (inside the mounted `models/` subdir), not elsewhere-then-move |
| Volume runs out of space despite models being smaller than the volume | Hugging Face's own download cache (`.cache/huggingface/download/...`) keeps a full duplicate blob alongside the materialized file | Delete that cache dir after each file finishes downloading; size the volume with headroom regardless |
| `ValueError: infer_schema(func): ... unsupported type list[int]` at ComfyUI startup | `cu124` PyTorch index is frozen at torch 2.6.0 (no new builds ever published there); that version is incompatible with `comfy-kitchen`'s custom-op schema | Check what torch versions each CUDA index tag *actually* still publishes (`curl https://download.pytorch.org/whl/<tag>/torch/` and grep versions) before assuming a "safe-looking" tag like `cu124` tracks current releases — some CUDA index tags are effectively abandoned. `cu126`+ was current at time of writing. |
| `AttributeError: module 'sys' has no attribute 'get_int_max_str_digits'` | Ubuntu 22.04's `python3.11` apt package is a **permanently frozen pre-release build** (`3.11.0~rc1`) that never gets point-release updates from the universe repo | Use Ubuntu 24.04's `python3` (3.12, properly maintained main-repo package) instead of chasing a specific Python version on 22.04 |
| `Cannot uninstall pip 24.0, RECORD file not found` | Ubuntu 24.04's distro-packaged pip can't uninstall itself in-place (Debian packaging convention) | `pip install --ignore-installed --upgrade pip` |
| `Failed to find C compiler` at first inference (triton JIT-compiling a kernel) | No compiler installed in the image at all | `apt install build-essential` |
| `gcc` invoked but still fails, `-I/usr/include/python3.12` in the failing command | Python headers (`Python.h`) missing — only the interpreter was installed, not dev headers | `apt install python3-dev` |
| `ImportError: cannot import name 'is_offline_mode' from 'huggingface_hub'` | A separate, later `pip install -r backend-requirements.txt` step **hard-pinned** `huggingface_hub==X.Y.Z`, silently downgrading/changing the version that an earlier-installed `transformers` actually needed | Don't hard-pin a shared transitive dependency in a second, separate pip invocation — use `>=` so pip's resolver doesn't clobber a version another package already needs. Two separate `pip install` calls do **not** jointly resolve dependencies the way one call would. |

General principle underneath all of these: **a "should just work" base image assumption is the thing most likely to be wrong.** Ubuntu LTS point releases, CUDA index tags, and distro Python packages all have their own quiet abandonment/versioning quirks that don't surface until something downstream needs a feature added after the point where that particular path stopped updating.

### CI pattern that works well
GitHub Actions + GHCR, no Docker Hub account needed: `docker/login-action` against `ghcr.io` authenticated with the workflow's own `secrets.GITHUB_TOKEN` (needs `permissions: packages: write` on the job) — zero manually-created secrets. `docker/build-push-action` with `platforms: linux/amd64` explicit (RunPod's fleet is x86_64; a Mac/ARM machine building without this flag produces an image that silently won't run there). Tag both `:latest` and `:v<run_number>` so every build is individually addressable. Path-filter the trigger (`backend/**`, `docker/**`, etc.) so docs-only commits don't waste a build.

---

## 4. Verified model integrations

### MiniMax H3 (video + audio, joint generation)
- Files (Comfy-Org repackaged): diffusion model `minimax_h3_fl2va_pruned_int8_convrot.safetensors` (~19.5GB), text encoder `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` (~15.7GB), video VAE + audio VAE (~5GB combined). Total ~40GB.
- **Correction (per Comfy Org's own official docs, 2026-08 — see §8): 80GB is not actually required.** I only empirically confirmed an 80GB A100 works; I never tested smaller and wrongly generalized "no smaller tier is viable." Comfy Org's own tested comfortable range is **a 30-series-or-newer GPU with 16GB+ VRAM**, using ComfyUI's dynamic VRAM offloading (not everything needs to fit in VRAM simultaneously — weights can be staged from system RAM as needed). Reference: ~9 min for 5s/480p on an RTX 3060, ~15 min for 5s on a 16GB card. Slower on a small card, but not impossible — and **generation time scales exponentially with pixel count, not linearly**, so re-estimate for the actual resolution wanted rather than scaling a 480p number linearly to 720p. Next time this model comes up, try a cheap 16-24GB tier first (e.g. RTX 4090, ~$0.34/hr) instead of defaulting to an 80GB A100 (~$1.19-1.39/hr) — the earlier conclusion was overcautious, not verified.
- Frame/duration grid confirmed independently by Comfy Org's docs too: 17k+5 frames at 24fps, up to ~15s max — matches what was found directly in ComfyUI's own MiniMax H3 node originally.
- Two distinct routes exist on Comfy Cloud under the same display title ("MiniMax H3: Text to Video"): OSS weights (`video_minimax_h3_*` internal name — what this skill covers) vs. a paid partner/API node (`api_minimax_h3_*`). Not relevant to a RunPod deployment, but useful context so a `search_nodes`-style result naming "MiniMax H3" isn't assumed to be the OSS route without checking the internal name/category.
- Native ComfyUI node: `MiniMaxH3ImageToVideo` — `first_frame` input is **optional**; omitting it gives pure text-to-video on the exact same node (no separate T2V node needed). Also `MiniMaxH3SigmaShift` (shift_video/shift_audio on the model), standard `KSamplerSelect`/`BasicScheduler`/`BasicGuider`/`SamplerCustomAdvanced` chain, `VAEDecodeAudio`+`VAEDecode`+`CreateVideo`(fps)+`SaveVideo`.
- Requires the custom Docker build in §3 (this is *the* model that needed it — `comfy-kitchen`+triton for int8/nvfp4 quantized ops).
- Does **not** run on Apple Silicon at all: MPS backend lacks the `_int_mm` op the quantized kernels need (`PYTORCH_ENABLE_MPS_FALLBACK=1` "fixes" it by falling back to CPU, which is catastrophically slow for a model this size — don't bother).

### Wan 2.2 (video only, no audio)
- **5B (TI2V) variant**: single diffusion model + text encoder + VAE. Only needs **~6-8GB VRAM/unified memory** — the one model in this whole body of work that genuinely fits comfortably on a 24GB Mac's memory budget (speed is a separate question, see §6).
- **14B (MoE) variant**: ~27B total across two 14B experts, selectively activated. VRAM ranges from ~8GB (GGUF-quantized, T5 offloaded to CPU) to ~54-65GB (full FP16 pipeline) depending on quantization.
- Native ComfyUI node: `Wan22ImageToVideoLatent` — takes `vae, width, height, length, batch_size, start_image (optional)`, outputs only a `LATENT` (no conditioning passthrough, unlike MiniMax H3's combined node — conditioning comes from separate `CLIPTextEncode` nodes wired straight into `KSampler`). Same optional-image pattern as MiniMax H3: omit `start_image` for text-to-video.
- **Frame-rate gotcha**: Wan 2.1 and the 14B variants run at **16fps**; the **5B TI2V variant specifically uses 24fps** — confirmed from the official reference workflow's `CreateVideo` node, not assumed from the wider Wan family's convention. Getting this wrong misjudges every duration calculation.
- **Frame-count constraint**: `length` (frames) must satisfy `(length - 1) % 4 == 0` (the VAE's 4× temporal compression). At 24fps: 121≈5.0s, 241≈10.0s, 481≈20.0s.
- Settings: `ModelSamplingSD3(shift=8)` (called `shift` even though it's a Wan-family node, not SD3), `KSampler(sampler_name="uni_pc", scheduler="simple", steps=20, cfg=5)`.
- Ships natively, no custom Docker image needed — RunPod's official ComfyUI template works as-is.

### ACE-Step 1.5 (music/song generation, text+lyrics-to-audio)
- **Turbo all-in-one** (`ace_step_1.5_turbo_aio.safetensors`, `Comfy-Org/ace_step_1.5_ComfyUI_files`, ~10GB): single-file checkpoint, `CheckpointLoaderSimple`, fastest (8 steps, cfg=1).
- **XL variants** (`acestep_v1.5_xl_{base|sft|turbo}_bf16.safetensors`, ~9.97GB each): a genuinely larger 4B-parameter diffusion transformer, not just a different schedule. Ships as **split files**, loaded via `UNETLoader`+`DualCLIPLoader`(both `qwen_0.6b_ace15.safetensors` + `qwen_4b_ace15.safetensors` together, `type="ace"`)+`VAELoader` — never `CheckpointLoaderSimple`. **`xl-sft`** = best final-quality audio (50 steps, cfg=7) — use this when the output actually matters, not `xl-turbo`, which trades quality for iteration speed. `xl-base` = more creative diversity, same 50-step cost.
- Node graph (both variants): `TextEncodeAceStepAudio1.5` (node_id literally has the `1.5` with its dot — `"TextEncodeAceStepAudio1.5"`, not a display-only label, it's the actual `class_type` string ComfyUI's API needs) → `ConditioningZeroOut` for the negative (**ACE-Step doesn't use a real negative prompt** — the official workflow zeroes a copy of the positive conditioning instead) → `ModelSamplingAuraFlow(shift=3)` on the model → `EmptyAceStep1.5LatentAudio(seconds=duration)` → `KSampler` → `VAEDecodeAudio` → `SaveAudioMP3` (inputs: `audio, filename_prefix, quality` — quality options `"V0"/"128k"/"320k"`).
- Ships natively, no custom Docker image needed.
- No verified VRAM figure exists specifically for XL from a primary source — a 24GB tier is a safe starting assumption (not confirmed as strictly necessary), worth experimenting downward once confirmed working.

### Gemma 4 (LLM — lyrics, tags, scene lists, general text generation)
- Ships **natively** in ComfyUI core (`comfy_extras/nodes_textgen.py`) as of the version checked — no custom LLM node pack needed for basic use.
- Node: `TextGenerate` — inputs `clip, prompt, image (optional), video (optional), audio (optional), max_length, sampling_mode ("on"/"off"), thinking, use_default_template`; when `sampling_mode="on"`, extra widgets appear: `temperature, top_k, top_p, min_p, repetition_penalty, presence_penalty, seed`. Genuinely multimodal — accepts image/audio/video as context, not just text.
- Loaded via the **generic** `CLIPLoader` (not a Gemma-specific loader), with `type="stable_diffusion"` — a generic/default value, not a Gemma-specific type tag, confirmed from the official template rather than assumed.
- Model: `Comfy-Org/gemma-4`, recommended file `gemma4_e4b_it_fp8_scaled.safetensors`, → `models/text_encoders/`.
- **Retrieving generated text via the HTTP API is the real gotcha**: `TextGenerate`'s `generated_text` output is a plain `STRING` — it does **not** automatically appear in a `/history/{prompt_id}` response the way `SaveImage`/`SaveAudio`/`SaveVideo` do, because those are `OUTPUT_NODE=True` nodes that explicitly populate a UI results dict, and a bare string output isn't. Fix: wire the output into a **`PreviewAny`** node (`source` input, accepts any type) — confirmed directly from its source (`comfy_extras/nodes_preview_any.py`) that it's `OUTPUT_NODE=True` and returns `{"ui": {"text": (value,)}}`. Retrieve via `history[prompt_id]["outputs"][node_id]["text"][0]`. Without `PreviewAny` (or an equivalent `OUTPUT_NODE`), a `TextGenerate` call's result is simply unreachable through the API — this is not documented anywhere obvious; found by reading the official reference workflow JSON's actual node wiring.

### LTX 2.3 (video, from Lightricks) — heavy, not yet integrated
- Diffusion model is **not** in the `Comfy-Org/ltx-2` repo (that repo only has LoRAs and text encoders) — it's in `Lightricks/LTX-2.3-fp8` (`ltx-2.3-22b-dev-fp8.safetensors`, part of a ~58.7GB repo) or the full-precision `Lightricks/LTX-2.3` (~157GB repo). This is a genuine **22-billion-parameter** model — closer to MiniMax H3's weight class than Wan 2.2's.
- Text encoder is Gemma-3-12B (`gemma_3_12B_it_fp4_mixed.safetensors` recommended, ~9.4GB) — a *different* Gemma usage than §4's Gemma 4 lyrics-writer; don't conflate the two just because both are named "Gemma."
- Total weights easily exceed 40GB — realistically needs an **80GB-class GPU** (A100), not a cheap consumer tier. This corrected an initial wrong assumption that LTX 2.3 would be a "Wan 2.2-cheap" video option.
- Native ComfyUI support: `comfy_extras/nodes_lt.py` (main video nodes: `EmptyLTXVLatentVideo`, `LTXVImgToVideo`, `ModelSamplingLTXV`, `LTXVScheduler`, `LTXVConcatAVLatent`/`LTXVSeparateAVLatent` for joint audio+video latents, dual-CFG guiders for separate video/audio CFG scales), `nodes_lt_audio.py`, `nodes_lt_upsampler.py`.
- Official template default: 1280×720, 97 frames (5s@25fps, the `(N×8)+1` frame-grid rule — **different from Wan's `(N×4)+1` rule**, don't mix these up), Euler, CFG 1.0.
- An "audio-reactive LoRA" exists (`100percentrobot/LTX-2.3-Audio-Reactive-LORA`, third-party, explicitly labeled "proof of concept" by its author) for syncing video motion to a music track's rhythm.
- **Status: researched, not built.** Given the cost/complexity, Wan 2.2 was chosen instead for the actual `music-video-generator` pipeline. Revisit if a project specifically needs audio-reactive motion or the largest available video model.

---

## 5. Cost estimation — do it with real numbers, not memory

- Always fetch **current** GPU pricing (`WebFetch` on `runpod.io/pricing`, or the `gpuTypes(input:{id}).lowestPrice` GraphQL query for live per-region stock+price) rather than recalling a number from earlier in a conversation or from training data — prices and stock both drift.
- Always fetch a **current** FX rate (`WebSearch` "USD to INR exchange rate today") when converting for a user who thinks in a different currency — don't reuse a rate from even a day earlier without rechecking.
- Give a **range**, not false precision, when the underlying generation-time figure is itself an extrapolation or a vendor's self-reported benchmark rather something you've measured yourself. Label which parts of an estimate are confirmed vs. extrapolated.
- `stockStatus: "Low"` is not `"None"` — a deploy attempt can still succeed; don't treat a discouraging stock indicator as a hard blocker without actually trying (see §2's retry-loop guidance).

---

## 6. Apple Silicon (Mac/MPS) — what actually works locally

- **"Fits in memory" and "runs fast" are different questions.** A model's VRAM/unified-memory footprint fitting under a Mac's total RAM says nothing about generation speed — MPS's raw compute throughput for diffusion workloads is well below even a modest dedicated NVIDIA GPU, independent of memory headroom.
- **Missing ops, not just slowness, can be a hard blocker**: MiniMax H3's int8 quantized kernels needed `aten::_int_mm`, which MPS doesn't implement at all — `PYTORCH_ENABLE_MPS_FALLBACK=1` "fixes" this by silently running that op on CPU, but for a 40GB model that makes a single generation take hours, not minutes. A memory-fits model can still be a non-starter if it depends on an op MPS lacks.
- **Wan 2.2 5B is the one video model found so far that both fits (6-8GB) and avoids exotic ops** (standard fp16, no int8/nvfp4 quantization) — the best local-Mac candidate identified, though never empirically timed on this specific machine; generation-time estimates for it locally (extrapolated from an RTX 4090 baseline plus a "several times slower" Mac multiplier) range from tens of minutes to hours depending on clip length, and could be worse than a simple linear extrapolation suggests since video-diffusion attention cost can scale worse than linearly with frame count.
- Recommended minimum unified memory tends to run higher than a model's "minimum to run at all" figure — e.g. LTX 2.3 wants 32GB+ recommended even though smaller quantizations are claimed down to less; treat vendor "minimum" and "recommended" numbers as different claims, not interchangeable.
- Practical takeaway: local Mac generation is worth it only when "free" matters more than "fast," and only for the lightest models (Wan 2.2 5B-class or smaller) — anything in MiniMax H3/LTX 2.3's weight class isn't viable locally at all, full stop, regardless of patience.

---

## 7. Architecture patterns that worked well across all four projects

- **Orchestration layer runs locally, generation runs on RunPod.** If a piece of code only makes outbound HTTP calls to a GPU pod's API and waits — it doesn't need a GPU itself. Running that piece locally (or on any $0 always-on machine) instead of on a third rented pod is free and exactly as functional. (`music-video-generator`'s FastAPI backend is the clearest example: it talks to two separate pods and never touches a GPU.)
- **Two-stage (audio pod + video pod) beats one big pod** when the two halves have very different GPU-tier needs — e.g. ACE-Step XL (24GB tier) + Wan 2.2 (24GB tier) run as two modest pods sequentially, rather than needing one pod sized for whichever half is heaviest simultaneously loaded.
- **One LLM call can drive multiple downstream stages.** Asking Gemma for lyrics + tags + a numbered scene list in one response (rather than a separate call per artifact) keeps the video's visual narrative thematically tied to the song's actual content, and halves the number of LLM calls needed.
- **Duration-driven clip counts should cycle/truncate a scene list, not require an exact match.** An LLM picking both a song's duration and its own scene count in the same breath won't reliably produce `scenes == duration/clip_length` — design for `scenes[i % len(scenes)]` instead of validating an exact count and failing.
- **ffmpeg concat should re-encode, not stream-copy**, when the input clips came from separate render calls (different ComfyUI jobs) — guarantees one consistent codec/timebase before a final mux, avoiding subtle A/V sync issues a `-c copy` concat can hit across heterogeneous sources.
- **Every project gets the same doc pair**: `README.md` (run instructions, deployment steps) + `docs/INTERNALS.md` (tech stack, file-by-file, a Mermaid pipeline diagram that renders natively on GitHub) + `docs/internals.html` (the same content as a designed standalone artifact, for a nicer read outside GitHub's rendering). This consistency makes every project's documentation predictable to navigate.
- **Test what you can without a live pod before spending money on one.** Every workflow-graph builder and text parser in these projects was unit-tested locally against realistic sample data (confirms JSON-serializability and correct parsing logic) before ever being pointed at a real pod — catches a whole class of bugs for $0.

---

## 8. Comfy Org's official tools — a complementary verification path

Comfy Org publishes their own Claude Code plugin: `github.com/Comfy-Org/comfy-skills/tree/main/claude-code` (plugin name `comfy-cloud`). Important distinction before using anything from it: **this connects to `cloud.comfy.org` — Comfy Org's own hosted/managed GPU service, a different product from self-hosting ComfyUI on a RunPod pod (what every project in this skill actually does).** Installing it doesn't change how RunPod deployment works; don't conflate "generate via Comfy Cloud" with "deploy to RunPod."

What it's genuinely useful for regardless of which infrastructure you deploy to: its MCP server exposes `search_nodes`, `search_models`, and `search_templates` tools, backed by Comfy Org's own live catalog — a faster, more authoritative alternative to manually fetching and reading ComfyUI source files (§1's rule) *when this plugin is installed*. `search_nodes` returns a node's exact inputs/outputs/category/pack; `search_templates` finds official pre-built workflow JSONs (the same kind of thing manually located via GitHub search throughout this skill so far). If this plugin is available in a future session, prefer it over a research agent fetching raw GitHub source for verifying a node's schema — same rigor, less manual work.

Its bundled command docs also carry genuinely useful, dated domain knowledge worth treating as a periodically-refreshed source, not a permanent one (their own docs say as much: *"model facts change over time... confirm current details rather than assuming they still hold"* — a good practice to apply to this whole skill file too, not just their content). Two concrete corrections this source produced for this skill:
- The MiniMax H3 VRAM correction in §4 above (80GB is not actually required — that was this skill's own overcautious generalization from one successful data point, not a verified floor).
- The general principle that video-diffusion generation time scales **exponentially with pixel count, not linearly** — factor this into any cost/time extrapolation across resolutions (§5), not just linear scaling by frame count.

---

## Keeping this current

This file lives at `~/Documents/ComfyWorkspace/comfyui-runpod-expert/SKILL.md`, symlinked into `~/.claude/skills/comfyui-runpod-expert/` so it loads as a real skill. To update it:

```bash
cd ~/Documents/ComfyWorkspace/comfyui-runpod-expert
# edit SKILL.md — add a row to a table, a new subsection, whatever's new
git add SKILL.md
git commit -m "describe what was learned"
git push
```

Repo: `github.com/vishalbhatt001/comfyui-runpod-expert`
