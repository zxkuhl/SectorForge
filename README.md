# SectorForge

**A workbench to build, decompile, and patch local AI models.** SectorForge treats a
model like a program you can crack open — decompile any **GGUF** or **LoRA** adapter
(including models already in **Ollama**) into a Ghidra-style workbench, patch its weights
without retraining, and build new models from modular "sector" adapters.

- **Decompile** — symbol tree of resolved sectors (tokenizer, embeddings, layers, output),
  a tensor listing with real byte offsets, a hex view, full-text search and cross-references.
- **Patch (no retrain)** — scale / zero / clip / set weights, raw-byte poke, swap tensors,
  replace-all references, **resize** (convert dtype, prune layers, fix `block_count`), and
  edit metadata, tokenizer and training data in place.
- **Build** — train independent LoRA **sectors** on one frozen base, pack the ones you want
  into a full model, export to GGUF. Plus one-shot **`patch-think`** (bake in visible
  `<think>` reasoning) and **`patch-vision`** (graft sight from a trained vision sector).
- **Run it your way** — a desktop app (**Electron**) or a headless **JSON API**; a **UI mode
  for humans** and a **headless mode for AI**, organized into reusable **projects**.

See **[Why it's built this way](#why-its-built-this-way)** for the one constraint that
shapes the design, and **[Disclaimer](#disclaimer)** before you patch or share a model.

## Install

```bash
python -m pip install -r requirements.txt
# for GGUF export:
powershell -ExecutionPolicy Bypass -File scripts/setup_llamacpp.ps1
```

QLoRA 4-bit needs an NVIDIA GPU + bitsandbytes. On CPU/AMD set `quant: none` in
`config/base.yaml` (full-precision LoRA — slower, more RAM, still works).

### AMD / Intel GPU on Windows (DirectML)

AMD RDNA1 cards (RX 5000, e.g. **RX 5600 XT**) have no ROCm support, so training
goes through **DirectML**:

```bash
python -m pip install -r requirements-amd.txt   # instead of requirements.txt
python -m sectorforge device                     # confirm it sees your GPU
```

`device` should report `backend: directml`. This path bypasses CUDA entirely —
**text, thinking, AND vision** sectors all run on your GPU via custom loops in
[train_dml.py](sectorforge/train_dml.py) (HF's trainer can't drive DirectML, so
we don't use it). Notes:
- **No 4-bit** — bitsandbytes is CUDA-only. `quant: none` (already set).
- **fp16 fits ~1.5B on 6 GB** (`Qwen2.5-1.5B-Instruct`, the default). Training is
  gradient-clipped so fp16 doesn't NaN. If it OOMs, drop `base_model` to
  `Qwen2.5-0.5B-Instruct`; if it goes unstable, set `compute_dtype: float32`
  (more stable, ~0.5B ceiling).
- Keep `batch_size: 1`, lean on `grad_accum`, `max_seq_len` ~1024. Slower than
  CUDA, but it's your card.
- **Vision on 6 GB is very tight** — use a *tiny* VLM
  (`HuggingFaceTB/SmolVLM-256M-Instruct` or `-500M`). A 2B VLM will OOM.

The only real ceiling is VRAM, not CUDA. Want 3B–8B? A few hours on a rented
NVIDIA box with the normal `requirements.txt` runs the same configs at `quant: nf4`.

## App, modes & projects

SectorForge ships as **one Python backend** with two front-ends and two modes.

**Desktop app (Electron).** We don't rewrite the ML/RE code in JS — the Electron shell
([electron/main.js](electron/main.js)) just spawns the Python backend and loads its UI:

```bash
npm install          # pulls Electron
npm start            # launches the desktop app (spawns `python -m sectorforge ui`)
```

**Two modes** (same backend):
- **UI (humans):** `python -m sectorforge ui` — the Ghidra-style workbench in a browser (or the Electron app).
- **Headless (AI/automation):** `python -m sectorforge serve --headless` — the JSON API only, no HTML. Plus every CLI command is headless by nature.

**Projects** — a saved workspace that bundles a task so you don't re-pass flags. A `train`
project holds a base + sectors/datasets (how we built Lynx); a `patch` project holds the
models you reverse-engineer and patch; `both` holds either:

```bash
sectorforge project create lynx --kind both \
    --base-config config/base-vision.yaml --sector config/sectors/vision-think.yaml \
    --target rms-think --notes "Kitty vision + RE/patch"
sectorforge project list
```

Projects live in `projects/<name>/project.json` and are served at `/api/projects`.

## Workflow

```bash
# 1. train sectors independently — each on a DIFFERENT dataset
python -m sectorforge train-sector config/sectors/example-text.yaml
python -m sectorforge train-sector config/sectors/example-vision.yaml

# 2. see what you've got
python -m sectorforge list

# 3. pack chosen sectors into one full model
python -m sectorforge pack --sectors lore-and-facts,screenshot-reader --out mymodel

# 4. export to GGUF (add --vision for a VLM: also emits an mmproj file)
python -m sectorforge export-gguf --packed out/packed/mymodel --out mymodel --quant Q4_K_M

# 5. update one slice later — retrain just that sector, re-pack. Nothing else moves.
python -m sectorforge train-sector config/sectors/example-text.yaml
python -m sectorforge pack --sectors lore-and-facts --out mymodel
```

### Per-sector datasets

Each sector yaml points at its own `dataset:`. Sector A can train on lore, B on
code, C on screenshots — totally different data, same base. That's the whole
point: a sector is `{ its data + its adapter }`.

### Patch a single fact

```bash
python -m sectorforge edit-fact --subject "Xinject" --relation "latest version" --value "v2.0"
```

Builds a tiny high-LR micro-adapter for just that fact; pack it in to apply. For
truly surgical weight edits use `--strategy memit` (needs EasyEdit + per-model
hyperparams — see `edit.py`).

## Decompile & patch a sector (no retrain)

A sector *is* a discrete file — a LoRA adapter (`adapter_model.safetensors`), a
bag of named float tensors. You can't byte-slice the *model* (knowledge is
distributed, see below), but you **can** crack open a *sector*, see exactly what
it changed, and edit its weights directly instead of retraining.

```bash
# read side: dataset it learned from, LoRA config, and the effective dW per
# layer/module (where the sector actually did its work)
python -m sectorforge decompile behavior

# deep-dive one tensor, with a raw hexdump + the file byte-offsets
python -m sectorforge decompile behavior --tensor layers.0.self_attn.q_proj.lora_B --hex 64
```

`patch` edits weights with no training and writes a new (or `--in-place`)
adapter that `pack` merges like any other sector:

```bash
# soften what layer 27's gate_proj learned (volume knob: 0=off, 0.5=soften, 1.5=amplify)
python -m sectorforge patch behavior --layer 27 --module gate_proj --scale 0.5 --out behavior-soft

# ablate a module across the whole adapter / zero one layer / clamp outliers
python -m sectorforge patch behavior --module o_proj --zero
python -m sectorforge patch behavior --layer 0 --zero
python -m sectorforge patch behavior --clip -0.01 0.01

# set one element; or a real byte-level replace of one whole tensor
python -m sectorforge patch behavior --set "layers.0.self_attn.q_proj.lora_B.weight[0,0]=0.5"
python -m sectorforge patch behavior --swap-bytes <src-tensor> <dst-tensor>   # same shape+dtype

# literal raw byte poke at a file offset (from `decompile --hex`) — warns loudly,
# because overwriting float bytes usually makes garbage unless you know the IEEE-754 bytes
python -m sectorforge patch behavior --in-place --poke "2476872=00000000"
```

Selectors (`--tensor` / `--layer` / `--module` / `--lora A|B`) AND together;
omit them all to hit the whole adapter. Out-of-place writes a fresh sector and
records `patched_from` + the op in the registry; `--in-place` backs up to
`adapter_model.safetensors.bak` first. Note `--scale F` on a module scales *both*
`lora_A` and `lora_B`, so the effective dW moves by F² — scale just `--lora B`
for a linear knob.

### Prebuilt models (Ollama / standalone GGUF)

`decompile` and `patch` also read **whole prebuilt models** — a `.gguf` path or
an Ollama model by name (it resolves the blob from `~/.ollama/models`):

```bash
python -m sectorforge decompile rms-coder-7b          # arch, metadata, tensors, quant mix
python -m sectorforge decompile rms-coder-7b --tensor output_norm.weight --hex 64
python -m sectorforge decompile path/to/model.gguf --json
```

Patching a GGUF always writes a **new file** (never Ollama's content-addressed
blob):

```bash
# edit metadata (rope, context length, name, chat params…) — structured header rewrite
python -m sectorforge patch rms-coder-7b --set-meta qwen2.context_length=8192 --out big-ctx.gguf

# edit float tensors (F32/F16/BF16 only) or raw bytes; quantized tensors are poke-only
python -m sectorforge patch model.gguf --tensor output_norm --scale 1.1 --out tuned.gguf
python -m sectorforge patch model.gguf --poke "453021472=0000803f" --out poked.gguf
```

Most of a quantized model is Q4_K/Q6_K blocks — those you can inspect by byte and
raw-`poke`, but not edit as floats (that needs dequant+requant). The F32 tensors
(norms, biases) *are* float-editable. Load a patched GGUF back with
`ollama create <name> -f Modelfile` (`FROM your-patched.gguf`).

### Workbench UI (Ghidra-style)

One screen to do all of the above — an object tree (sectors + Ollama models), a
**Symbol Tree** of resolved sectors, a tensor **Listing** with offsets, a
**Decompile** pane (metadata / config / effective dW), a **Bytes** hex view, and
a patch toolbar:

```bash
python -m sectorforge ui          # opens http://localhost:8777
```

Pure stdlib server; it calls the same functions the CLI does.

### Analysis & Symbol Tree

Opening an object runs a quick analysis that resolves its logical **sectors** and
shows them in a Symbol Tree, each at its start offset — double-click to jump there:

- **GGUF:** Metadata (header) · Tokenizer (tokens / merges / token_type, with their
  byte offsets in the header) · Embeddings · Layers → `blk.0…N` → each block's
  tensors · Output head.
- **LoRA:** Config · Dataset · Layers → `layer N` → `self_attn` / `mlp`.

When analysis finishes, a prompt offers **"Go to weights →"**, which jumps to the
start of the tensor-data section (the first weight), Ghidra's "go to entry point".

### Search & replace (Ghidra-style)

Find anything and **double-click a result to jump to it** (selects the Listing
row + loads the Bytes view, scrolling to the exact offset for byte hits):

```bash
python -m sectorforge search behavior q_proj --scope tensors
python -m sectorforge search rms-coder-7b rope --scope metadata
python -m sectorforge search rms-coder-7b def  --scope strings     # tokenizer tokens/merges
python -m sectorforge search behavior "RMS"    --scope dataset
python -m sectorforge search rms-coder-7b 0000803f --scope bytes   # hex in the data region
```

"Replace all references" — substring across all GGUF string metadata, or an
equal-length raw byte replace (header is never touched; Ollama blobs never
clobbered):

```bash
python -m sectorforge replace rms-coder-7b --meta "Qwen" "RMS" --out rebranded.gguf
python -m sectorforge replace model.gguf   --bytes 0000803f 00000040 --out doubled.gguf
```

Both are in the UI too: the search bar + Search Results pane, and the ⇄ Replace bar.

### Cross-reference, sector-patch & resize (right-click)

Right-click any node in the Symbol Tree (or use the CLI):

```bash
# cross-reference a sector: tensor count, params, bytes, quant dominance, dW
python -m sectorforge xref rms-coder-7b blk.27.
python -m sectorforge xref behavior layers.27.

# patch a WHOLE sector at once (same ops as `patch`, selected by prefix)
python -m sectorforge patch rms-coder-7b --tensor blk.27. --zero
python -m sectorforge patch behavior --tensor layers.27. --scale 0.5
```

**Change the actual model size** — convert dtypes or prune tensors, which rewrites
the whole file (new offsets, new size):

```bash
python -m sectorforge resize model.gguf  --tensor output_norm --convert F16 --out smaller.gguf
python -m sectorforge resize model.gguf  --tensor blk.27.     --prune       --out trimmed.gguf
python -m sectorforge resize behavior    --tensor layers.27.  --convert F16
```

Because these change the file size, the UI shows a confirm first:

> **⚠ WARNING** — to complete this action the file size must change, so we have to
> patch the weights and rebuild the model file. *P.S: This may cause weight
> corruption.*

Convert only works on float tensors (F32/F16/BF16); quantized tensors (Q4_K/Q6_K…)
can't be dtype-converted without a requantizer, so they're prune-only. Pruning
**whole `blk.N` blocks** renumbers the survivors contiguously and decrements
`<arch>.block_count`, so a trimmed model still loads (a shallower model).

### Open & read the text data (tokenizer / training data)

Right-click **Tokenizer** tokens/merges/chat_template (GGUF) or the **Dataset**
(LoRA) node → **Open & read** to view — and edit — the actual text:

- **Training data** (LoRA `.jsonl`): paginated rows, filter, and edit/add/delete a
  row. These write the dataset file directly (backed up to `.jsonl.bak`); no weights
  touched.
- **Tokenizer tokens / merges** (GGUF): paginated (200 at a time of ~152k) with
  filter. **Stage** several edits (they show amber with a ● marker) and hit
  **Apply N edits** to write them all in a single rewrite — one pass over the file,
  not one per token. Discard clears the queue.
- **chat_template / string metadata** (GGUF): full editable text.

Editing GGUF text changes the file size, so it writes a new `.gguf` and shows the
weight-corruption warning first.

## Capture a model's thinking (`patch-think`)

Patch `<think>…</think>` straight into a prebuilt model so every reply opens a
reasoning block you can capture — it edits the GGUF chat template to prefill
`<think>` at the assistant turn:

```bash
python -m sectorforge patch-think rms-coder-7b --out rms-think.gguf
# then, since Ollama uses the gguf's embedded template:
printf 'FROM rms-think.gguf\n' | ollama create rms-think -f -
```

Now every response is `<think> …reasoning… </think> …answer…`, and
`sectorforge.think.split_thinking(text)` pulls the two apart. In the workbench
it's a right-click on the **Tokenizer** node → *Patch: force `<think>` reasoning*
(it rewrites the file, so it shows the size-change warning first).

Note this captures the reasoning the model *narrates*, which isn't guaranteed to
be the computation it actually ran (CoT can be post-hoc). To bake genuine
reasoning into the weights instead, train a thinking sector (below).

## Thinking / reasoning

Make a sector that emits visible reasoning. Set `style: reasoning` and give it
data with `thinking` + `answer` — training folds them into the assistant turn as
`<think>…</think>` then the answer, so the reasoning lives in the weights.

```bash
python -m sectorforge train-sector config/sectors/example-thinking.yaml
python -m sectorforge pack --sectors reasoning --out mymodel
```

Best results: distill `<think>` traces off a strong reasoning model
(DeepSeek-R1 etc.) into the `{prompt, thinking, answer}` format, R1-Distill style.

At inference, decide whether to show or hide the reasoning:

```python
from sectorforge.think import split_thinking, generate
full = generate(model, tok, "Is 91 prime?", show_thinking=True)   # includes <think>
s = split_thinking(full)
print(s.thinking)   # the reasoning  -> render as a collapsible panel
print(s.answer)     # the final answer only
```

Most chat UIs (and llama.cpp) already special-case `<think>` and render it as a
collapsible "Thinking" section. For reasoning that *emerges* from reward rather
than imitation, that's the GRPO/RL track (heavier — ask and I'll add it).

## Vision

Point `base_model` at a VLM (Qwen2-VL, Llama-3.2-Vision, LLaVA, moondream) and
use a `modality: vision` sector. Training LoRA-tunes the language side (and the
projector if `train_projector: true`) on image+text pairs. Export emits the
`.gguf` **and** an `mmproj-*.gguf` (the vision tower) — llama.cpp needs both to
run a VLM. Some VLMs need their model-specific mmproj converter; export prints
the exact fallback if `--mmproj` isn't supported for your arch.

## Why it's built this way

The tempting design — split the weight file into fixed-size chunks and retrain
"chunk 32" — doesn't work: a transformer's knowledge is **distributed** across
all layers, not stored in contiguous byte ranges. There's no "file 32 = the
outdated fact." So instead of slicing bytes, sectorforge slices along the axis
that *is* modular:

| your idea | what actually delivers it |
|-----------|---------------------------|
| multiple models, shared weights | LoRA adapters over one frozen base |
| train a part on weak hardware | QLoRA (4-bit base + small adapter) |
| retrain the outdated part, repack | retrain that adapter → `pack` |
| surgically fix one fact | `edit-fact` (micro-adapter) / MEMIT |
| "multiple models but one" (literal) | Mixture-of-Experts (separate track) |
| split into files that recombine | GGUF is the final inference format |

GGUF is an **inference** format — you train in safetensors and export to GGUF at
the end (step 4). That's why training never touches GGUF directly.

## Layout

```
config/base.yaml            shared base, quant, default LoRA, paths
config/sectors/*.yaml       one per sector: dataset + modality + hyperparams
sectorforge/
  train.py    QLoRA training (text via TRL SFT, vision via processor collator)
  pack.py     merge adapters into a full model (merge_and_unload)
  export.py   convert + quantize to GGUF (+ mmproj for vision)
  edit.py     single-fact patch (micro-adapter, or MEMIT hook)
  decompile.py open a sector OR prebuilt GGUF: data, config, weights, offsets,
              + analysis (resolve sectors -> symbol tree, weights entry)
  search.py   find in tensors/metadata/strings/dataset/bytes (jump-to-hit)
  textio.py   read/edit text data: training .jsonl rows, tokenizer tokens/merges,
              GGUF string metadata
  patch.py    edit weights with no retrain (scale/zero/clip/set/poke, + GGUF meta,
              + replace_meta / replace_bytes, + resize_model: convert/prune/block-count)
  gguf_io.py  read/locate GGUF models (incl. Ollama blobs), dequant, serialize
  ui.py       Ghidra-style browser workbench for decompile + patch
  project.py  projects (train / RE-patch workspaces)
  registry.py sector manifest (dataset, version, adapter path)
electron/                   desktop shell that spawns the Python backend
out/                        adapters/, packed/, gguf/, registry.json
projects/                   saved project workspaces
```

## Disclaimer

SectorForge is a tool for legitimate model engineering, research, and interoperability.
It inspects and edits model **files you own or are otherwise allowed to modify** — it does
not break DRM, bypass access controls, or circumvent any technical protection measure.

**You are responsible for how you use it.** A model may be governed by its own license and
by the laws of your jurisdiction; modifying, quantizing, grafting capabilities onto,
redistributing, or deploying a model can be subject to those terms. Before you patch or
share a model, make sure you have the right to do so, and do not use SectorForge to create
or distribute harmful, infringing, or deceptive models.

The software is provided **"as is", without warranty of any kind**. To the maximum extent
permitted by law, the authors and copyright holders accept **no liability** for any claim,
damages, or other consequences arising from use of the software or from modifying an AI
model with it. The choices you make with this tool, and their outcomes, are yours. See
[LICENSE](LICENSE).

## License

[MIT](LICENSE) © 2026 RMS Studios. Permissive — use, modify, and redistribute freely —
with **no warranty and no liability** on the authors (see the clause in [LICENSE](LICENSE)
and the [Disclaimer](#disclaimer) above).
