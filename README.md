
# ComfyUI Dynamic Prompting Simplified

A minimal **dynamic prompting + mirrored wildcards** node for ComfyUI.

## Overview
- `{a|b|{c|d}}` nested choice expansion (deterministic by seed).
- `__name__` loads `wildcards/name.txt`.
- `__name-mir__` strictly loads `wildcards/name-mir.txt`.
- In **negative** phase, if `__name__` is used and `name-mir.txt` exists, it auto-mirrors to **all options except the chosen one**.
- Deterministic selection per seed.

## Install
1. Copy this folder to: `<ComfyUI>/custom_nodes/ComfyUI-DynPromptSimplified`
2. (Optional) Put your wildcard files in `ComfyUI-DynPromptSimplified/wildcards/` or set `wildcard_dir` to any folder.
3. Restart ComfyUI.

## Node
**DynPrompt Expand (deep-mirrored)**
- Inputs: `text`, `negative`, `seed`, `variety`, `wildcard_dir`, `auto_neg_from_mir`
- Outputs: expanded `text`, expanded `negative`

## Node Inputs
- text: positive prompt
- negative: negative prompt
- seed: this is used to determine how the node will select tags
- variety: int between 0-100; creates extra randomness 'lanes' [Variety](#-variety-extra-randomness-lanes)
- wildcard_dir: folder with your wildcard .txt files; leave blank to use the `wildcards/` folder next to the node
- auto_neg_from_mir: automatically adds -mir tags to negative prompt without explicitly needing to add it to the negative prompt
<br>
Wire the outputs into your usual `CLIP Text Encode` node <br>
You can preview what the output is with the "preview any" node in utils
<br>
 <img src="https://raw.githubusercontent.com/RegulusAlpha/ComfyUI-DynPromptSimplified/refs/heads/main/images/1.png" alt="Preview 1" width="800"/>
<br>
## ✨ Features

- **Nested choices** with `{option1|option2|{nested1|nested2}}`
- **Wildcard expansion** with `__name__` → expands from `wildcards/name.txt`
- **Line-separated wildcard files** that may themselves contain braces and wildcards
- **Mirrored wildcards** with `__name-mir__`:
  - Positive prompt gets the **chosen option**
  - Negative prompt gets **all the other options, comma-separated**
  - Strict resolution: `__name-mir__` reads **only** `wildcards/name-mir.txt` (no fallback to `name.txt`)
- **Deterministic behavior** using the current generation seed
- Works for both **positive** and **negative** prompts
- Expanded prompts are saved into PNG metadata

  ## 📑 Wildcard Files

- Located in the `wildcards/` directory (configurable).
- Each line is one possible expansion.
- Blank lines and lines starting with `#` are ignored.
- Lines may contain further braces `{}` and wildcard calls.

**Example: `wildcards/hats.txt`**
```
{red hat|blue hat|{green hat|yellow hat|{black hat|gold hat}}}
beret
top hat
```

## 🔄 Mirrored Wildcards

- File name: `name-mir.txt` (called via `__name-mir__`).
- Ensures complementary picks between positive/negative prompts.

**Example: `wildcards/hats-mir.txt`**
```
{red hat|blue hat|green hat}
{tall hat|short hat|medium hat}
```

- Positive prompt: `portrait, __hats-mir__`
- Negative prompt: `lowres, __hats-mir__`
  - If pos → `red hat`, neg → `blue hat, green hat`
  - If pos → `short hat`, neg → `tall hat, medium hat`

This ensures that the **negative prompt excludes the token chosen in the positive prompt.**

### 🧩 Nested `-mir` behavior (auto-discovery)

Mirrored tokens can be **nested inside other wildcards**. If a wildcard you use in the **positive** prompt expands to another token like `__foo-mir__`, the extension can **automatically inject the mirrored complement** into the **negative** prompt (so you don’t have to add `__foo-mir__` manually).

- This requires the checkbox in the UI:  
  **“Automatically mirror -mir wildcards without explicitly adding them to the negative prompt.”** (enabled by default)
- The auto-inject only happens if the negative prompt does **not** already contain that `__*-mir__` token.
- Resolution is strict: `__foo-mir__` reads `wildcards/foo-mir.txt` only.

**Example (nested):**
```
wildcards/outfits.txt
---------------------
__hats-mir__, {casual|formal}

wildcards/hats-mir.txt
----------------------
{red hat|blue hat|green hat}
```

Usage:
- Positive: `portrait, __outfits__`
- Negative: *(leave blank or put your usual negatives)*
<br>
- add node from: add node -> prompt -> DynPrompt Expand (deep-mirrored)
<br>
 <img src="https://raw.githubusercontent.com/RegulusAlpha/ComfyUI-DynPromptSimplified/refs/heads/main/images/2.png" alt="Preview 1" width="500"/>
<br>

Behavior:
- The positive prompt expands `__outfits__` → which contains `__hats-mir__`.
- With the checkbox enabled, the extension auto-injects `__hats-mir__` into the **negative** and expands it there as the comma‑separated “other” options.
- If the seed picks **`red hat`** for positive, the negative gets **`blue hat, green hat`** automatically.

## 🎲 Variety (extra randomness lanes)

**What it is:** An integer knob that creates alternate, reproducible randomness *lanes* without changing your seed or prompt.

**Deterministic:** With the same `(seed, variety, prompt, files)`, you get the same positive & negative outputs every time.

**Independence from fixed text:** Only decision points advance randomness—each `{…}` block and each `__wildcard__` / `__*-mir__`. Fixed words/commas/spaces do **not** affect picks.

**How it works (under the hood):**
- Choices use `choice#0, choice#1, …`
- Wildcards use `wild#0, wild#1, …` (also salted with the token name)
- We hash `(seed, counter[, token], variety)` to pick an index.

**When to use:** Keep `seed` fixed and sweep `variety = 0,1,2…` to explore different, reproducible variants from the same prompt.

**Example**
```text
Prompt: __color__, {A|B|{C|D}}, __pose-mir__
```
(seed=123, variety=0) → picks one set of branches
(seed=123, variety=1) → different set of branches (still reproducible)

## ⚙️ Usage

- Write prompts as usual with `{}` and `__wildcards__`.
- Wildcards are recursively expanded.
- Mirrored wildcards respect complement logic.

**Prompt Example:**
```
Positive: portrait, {cinematic|studio|outdoor}, __hats__, soft lighting
Negative: lowres, bad anatomy, __hats-mir__
```

## Logic

**Determinism & Counters**
- Randomness depends **only** on decision points:
  - Each non-nested `{…}` is the next **choice#N**.
  - Each `__token__` (including `__*-mir__`) is the next **wild#M**.
- For each pick we hash `(seed, counter[, token], variety)` → index.
- Fixed text (words/commas/spaces) does **not** affect counters or picks.
- `variety` is an extra integer salt to create reproducible “lanes”.

**Expansion Order (Positive)**
1) **Braces-first pass** (repeat until none left, up to 1024 blocks)  
   Collapse the left-most non-nested `{…}` using **choice#N**.
2) **Wildcard pass** (left → right)  
   Replace each `__token__` using **wild#M**:
   - `__name__` → read `wildcards/name.txt` (supports subfolders like `a/b.txt`).
   - `__name-mir__` → **strict** read `wildcards/name-mir.txt` only (no fallback).
   - If a chosen line itself contains braces/wildcards, it’s injected and will be handled by subsequent passes.
   - Missing file ⇒ token disappears (empty).
3) **Braces-second pass**  
   Collapse any `{…}` that arrived from wildcard lines.
4) **Cleanup**  
   Normalize commas/spaces.

**Deep Mirroring (Negative complement)**
- Your user Negative is expanded **normally** (same algorithm as Positive: braces → wildcards → braces), except:
  - An explicit `__name-mir__` in the Negative becomes **every option the Positive did not pick** for that token (never the picked one).
  - If the Positive never used that token, it becomes **all** of its options.
- If `auto_neg_from_mir` is **ON**, we run a **mirror pass** over the Positive prompt:
  - Re-run the same brace + wildcard passes with the same `(seed, variety)` and counters.
  - For every `__*-mir__` encountered (even if nested inside other wildcards):
    - Compute the chosen option.
    - Add **all non-empty alternatives except the chosen one** to a bag.
    - Special case: if the chosen alternative is **empty** (from `{option|}`), add all **non-empty** options.
- Merge: `final_negative = dedup(user_negative_expanded + mirrored_bag)`.

**File Semantics**
- **Single-line files**:
  - Non-mir: if the whole line is one block like `{a|b|{c|d}|}`, split by **top-level** `|` (empties kept as `""`). A line like `{a|b} dress, {c|d}` is one option and its braces are picked normally.
  - `*-mir`: braces are **flattened** to leaf options (`{a|b|{c|d}}` → `["a","b","c","d"]`, `{red|blue} hat` → `["red hat","blue hat"]`) so “all-except-chosen” is well-defined.
- **Multi-line files**: each non-empty, non-comment line is an option (lines may contain braces/wildcards).
  - `*-mir` files whose lines contain braces are flattened the same way, so the Negative gets plain options instead of raw `{…}`.
- **Path safety**: allows subfolders (`a/b`), dotted names (`style.v1`); blocks traversal (`..`).

**Pseudocode (conceptual)**
```
    counters = {choice: 0, wild: 0}

    def pick_choice(options):
        i = hash(seed, f"choice#{counters.choice}", variety) % len(options)
        counters.choice += 1
        return options[i]

    def pick_wild(token, options):
        i = hash(seed, f"wild#{counters.wild}:{token}", variety) % len(options)
        counters.wild += 1
        return options[i]

    # Positive:
    text = collapse_braces(text, pick_choice)        # pass 1
    text = expand_wildcards(text, pick_wild)         # pass 2
    text = collapse_braces(text, pick_choice)        # pass 3

    # Negative:
    neg  = collapse_braces(neg, pick_choice)
    neg  = expand_wildcards(neg, pick_wild)
    neg  = collapse_braces(neg, pick_choice)

    # Deep mirror additions:
    mirror_bag = run_positive_again_collecting_all_except_chosen_for('*-mir')

    final_negative = dedup_join(neg, mirror_bag)
```
**Key Guarantees**
- Same `(seed, variety, prompt, files)` ⇒ same Positive & Negative.
- Adding/removing fixed text does **not** change picks.
- Changing decision points (adding/removing a `{…}` or `__wildcard__`) changes the sequence of draws (by design).


## Notes
- Missing wildcard files resolve to empty strings.
- Expansion is capped to prevent runaway recursion: each brace pass resolves up to 1024 `{…}` blocks and the wildcard pass (including the mirror pass) up to 1024 `__tokens__`. Anything past the cap is left in the prompt as-is.
- With `auto_neg_from_mir` ON, mirrored exclusions are appended to the negative whether or not you typed a negative.

## Changelog
See [CHANGELOG.md](CHANGELOG.md).

License: GPLv3 (same as the original)
