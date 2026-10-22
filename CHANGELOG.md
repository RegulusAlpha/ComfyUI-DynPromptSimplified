# Changelog

### 1.0.1 — 2026-10-22 — Raise expansion limits, mirroring fixes
**Why**: large wildcard lines (and wildcards nested inside them) can hold more than 64 `{…}` blocks; anything past the 64th was left in the prompt as raw braces.

**What changed**
- Each brace pass now resolves up to **1024** `{…}` blocks (was 64).
- The wildcard pass now expands up to **1024** `__tokens__` (was 128).
- Fixed explicit `__name-mir__` in the Negative: it was expanded as a normal pick, landing on the **same** option as the Positive (so the Negative excluded exactly what the Positive asked for). It now becomes every option the Positive did not pick, or all options if the Positive never used that token.
- The deep-mirror pass is capped at the same 1024 tokens.
- Fixed single-line wildcard files such as `{a|b} dress, {c|d}` being split into broken options (`b} dress, {c`). Only a line that is one whole `{…}` block is split into choices; anything else is one option.
- Fixed raw `{…}` text reaching the Negative from multi-line `*-mir` files: lines with braces are now flattened into plain options, like single-line `*-mir` files.
- Fixed flattening dropping the space next to a brace (`{red|blue} hat` gave `redhat`).
- Fixed leftover `, ,` when three or more commas ended up in a row. It had no cap, so a self-referencing wildcard could hang the node when `auto_neg_from_mir` was on.
- Prompts that already fit under the old limit expand exactly as before (same seed + variety ⇒ same output).

### 2025-08-24 — Counter-Based Randomness & “Variety” lanes
**Why**: fixed tokens could cause prompt to converge.

**What changed**
- **Decision-only RNG**: Random choices now depend **only** on decision points (each `{…}` block and each `__wildcard__` / `__*-mir__` call), not on surrounding fixed text.  
  - Internally we hash `(seed, decision_counter[, token], variety)` where counters advance as we hit choices/wildcards left→right.
  - Adding/removing fixed words, commas, or spacing no longer shifts picks.
- **Deterministic lanes via `variety`**: New integer input that adds an extra salt.  
  - Same `(seed, variety, prompt, files)` ⇒ same outputs.  
  - Sweep `variety = 0,1,2…` to get reproducible alternates without touching seed or prompt.
- **Deep mirroring (strict)**: `__foo-mir__` contributes mirrored exclusions to Negative even when **nested** inside other wildcards, and **only** reads `foo-mir.txt` (never falls back).
- **Empty-branch support in `*-mir.txt`**: `{option|}` is honored.  
  - If Positive picks `option`, Negative adds nothing for that token; if Positive picks the empty branch, Negative adds the non-empty options.
- **Single-line `*-mir.txt` with nested braces**: `{1|2|{3|4}}` is flattened to leaves so mirroring can compute “all-except-chosen” correctly.

**Behavior notes**
- Fixed text no longer influences randomness. Only the **number/order** of decision points matters.  
- Changing decision points (e.g., adding a `{…}` or another `__wildcard__`) will change the sequence of draws—as expected.  
- Deep mirror uses the same counters as Positive, ensuring the Negative always gets the correct complement.

**Example**
```text
Prompt: __color__, {A|B|{C|D}}, __pose-mir__
Same seed, same variety → same picks (pos & neg)
Change only fixed words → picks stay identical
Bump variety (e.g., 0→1) → different, reproducible picks
```
