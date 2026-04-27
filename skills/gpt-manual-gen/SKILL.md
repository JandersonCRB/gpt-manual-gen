---
name: gpt-manual-gen
description: Use when generating images for a UI being built — icons, illustrations, hero images, avatars, photos, or any visual asset that needs creating. Drafts ChatGPT (GPT Image 2) prompts with reference-image awareness, hands off to the user for manual generation, then verifies the saved images programmatically and visually.
---

# Manual GPT Image 2 generation workflow

Drives a human-in-the-loop workflow: Claude plans descriptive image prompts, the user runs them in ChatGPT (GPT Image 2) and saves the results, then Claude verifies before continuing development.

## When this skill activates

- The current task involves a UI/website that needs images: icons, illustrations, hero images, avatars, photos, decorative elements.
- The user invokes `/gpt-manual-gen`.
- The user explicitly asks for image generation.

## When this skill does NOT activate

- Images already exist in the project and only need referencing or styling.
- The task is to integrate already-generated images (just `<img>` placement, no generation).
- The user wants images generated via API instead of manually (this skill is explicitly manual).

If activated and no images are actually needed — for example, the task is styling, layout adjustment, or integrating already-existing images — bail early with: "No image generation needed for this task." Do not write any files.

## Phase 1: Survey

Identify what images the current UI work needs. For each, note:
- A clear purpose (e.g., "search button icon", "hero illustration for landing page").
- A target file path (e.g., `assets/icons/search.png`). Choose paths that match the project's existing convention.
- The category (icon, illustration, photo, hero, etc.) — this informs prompt style.

Then scan for reference images. Search these directories from the project root, in this order:
1. The target output directory of each new image (siblings).
2. `public/`, `assets/`, `src/assets/`, `static/`.

Match these extensions: `.png`, `.jpg`, `.jpeg`, `.webp`, `.svg`.

For every reference found, open it with `Read` (Claude is multimodal) and extract visual identity cues:
- Palette (specific hex values where possible).
- Stroke weight and treatment (rounded vs. squared caps, uniform vs. tapered).
- Corner radius style.
- Illustration approach (flat, gradient, line art, isometric, photographic).
- Lighting and mood (for photos/illustrations).
- Subject framing and padding.

Pick the strongest reference per new image — prefer same-category siblings (e.g., another icon for an icon, another hero for a hero). If multiple candidates fit, pick one and note why.

If no reference images are found, the workflow proceeds — Phase 2 prompts will lean harder on textual style description.

## Phase 2: Plan

### Check for stale state

If `.gpt-manual-gen/prompts.md` already exists at the project root, do NOT overwrite. Read it first and ask the user:

> "A previous gpt-manual-gen run is in progress at `.gpt-manual-gen/prompts.md`. Resume from it (verify the existing entries) or start fresh (delete and replan)?"

Wait for the answer. Map the user's reply:

- **Resume** signals: "resume", "continue", "keep it", "where we left off", "verify what's there". Skip ahead to Phase 4.
- **Start fresh** signals: "fresh", "restart", "redo", "delete", "replan", "start over". Delete `.gpt-manual-gen/` and continue below.
- **Unclear**: ask again with explicit options: "Reply 'resume' to verify the existing images, or 'fresh' to delete and replan."

### Update `.gitignore`

If `.gitignore` exists in the project root, check whether it contains `.gpt-manual-gen/`. If not, append it (with a leading newline if needed). If `.gitignore` does not exist, create it with the single line `.gpt-manual-gen/`.

### Write `prompts.md`

Create `.gpt-manual-gen/prompts.md` at the project root. Use exactly this format (the verification script depends on it):

````
# GPT image prompts

Generated <YYYY-MM-DD> — generate each image in ChatGPT (GPT Image 2),
save to the listed path, then say "continue" to verify.

## 1. <target path>
- **Format:** <one of: PNG (transparent background) | PNG | JPG>
- **Native size:** <one of: 1024×1024 | 1792×1024 | 1024×1792>
- **Reference to attach:** <reference path>
- **Status:** pending
- **Prompt:**
  > <descriptive prompt, multiple lines OK, each line prefixed with `> `>

## 2. <target path>
...
````

**Field rules:**

- **Heading:** sequential 1-indexed integer + path. Path is relative to project root. Example: `## 3. assets/icons/menu.png`.
- **Format:** must be one of those three exact strings. Use `PNG (transparent background)` for icons and any image with transparency. Use `JPG` only for photos/heroes that don't need transparency.
- **Native size:** must be one of GPT Image 2's native sizes. Use `1024×1024` for square (icons, avatars, square illustrations). Use `1792×1024` for wide (heroes, banners). Use `1024×1792` for tall (mobile splash, vertical illustrations). Resizing for final use is the project build pipeline's job — do not request non-native sizes.
- **Reference to attach:** if a reference was found in Phase 1, write its project-relative path. If no reference applies, omit the entire line (do not write `null` or leave empty).
- **Status:** always `pending` for newly written entries.
- **Prompt:** descriptive and specific. Cover: subject (what's depicted), composition (framing, padding, focal point), palette (specific colors, ideally hex), style (matching the reference's stroke/treatment/mood), and any explicit constraints (transparent background, no text, etc.). Block-quote each line. The more concrete and visual, the better — GPT Image 2 rewards detail.

After writing the file, print to chat:

> "<N> images planned in `.gpt-manual-gen/prompts.md` — generate them in ChatGPT, then say `continue` when done."

Then stop. Do not start work on anything else.

## Phase 3: Handoff

Wait for the user's signal (e.g., "continue", "done", "ready", "go"). While waiting, do not generate code, do not ask follow-up questions about the broader task — the next move belongs to the user.

## Phase 4: Verify

Run on the user's continuation signal.

### Step 4a: Programmatic check

Run the verification script. Locate it by trying these paths in order:

1. `scripts/verify-images.mjs` (relative — works when running from inside the plugin's own repo or a project that has cloned the plugin)
2. `${CLAUDE_PLUGIN_ROOT}/scripts/verify-images.mjs` — works when the plugin is installed via `/plugin install` if Claude Code exposes that env var to the shell
3. `~/.claude/plugins/gpt-manual-gen/scripts/verify-images.mjs` — common install location fallback

Run it as:

```
node <resolved-path> .gpt-manual-gen/prompts.md
```

If none of the paths resolve, ask the user where the plugin is installed before continuing.

The script:
1. Parses `prompts.md`.
2. For each entry whose Status is `pending` or `failed: ...`, runs file-existence, format-extension, dimension, and file-size checks.
3. Writes results back into `prompts.md` as updated `Status:` lines (`verified` or `failed: <reason>`).
4. Prints a per-entry pass/fail summary and exits 0 (all passed) or 1 (some failed).

Read the updated `prompts.md` after the script runs. Note which entries are now `verified` and which are `failed: ...`.

### Step 4b: Visual check

For every entry that the script JUST promoted from `pending` or `failed:` to `verified` in this Phase 4 cycle (and only those — entries the script marked `failed` skip this step; entries that were already `verified` before this cycle started are not re-opened), open the image with `Read` and judge:

- **Subject match:** does the image depict what the prompt asked for?
- **Style match:** if a reference was attached, does the style align with the reference (stroke, palette, treatment)?
- **Artifacts:** any glaring problems — extra elements, wrong colors, distorted geometry, unwanted text, watermarks?

If an image fails visual review, manually update its `Status:` line in `prompts.md` from `verified` to `failed: <specific reason>` (e.g., `failed: subject is a binoculars icon, prompt asked for magnifying glass`). Use the Edit tool.

### Step 4c: Report

After both steps, count:
- Entries with Status `verified` → passed.
- Entries with Status starting `failed:` → failed.

If all entries are `verified`: announce success and proceed to Phase 5.

If any are `failed:`: report each with its specific reason. Ask the user:

> "<N> images need regenerating: [list]. Regenerate those in ChatGPT and save back to the same paths, then say `continue` to re-verify. (You can also ask me to refine the prompt for any of them.)"

Wait for the user. On the next continuation signal, repeat Phase 4 — but only `pending` and `failed:` entries are reprocessed. Already-`verified` entries are skipped at every level (no script work, no token cost).

## Phase 5: Cleanup

Once every entry's Status is `verified`:

1. Delete the `.gpt-manual-gen/` directory entirely.
2. Announce: "All images verified — continuing with [original task]."
3. Resume the UI work that triggered this skill.

The verified images themselves at their final paths are the artifacts — `prompts.md` is no longer needed.

## Edge case checklist

- **No images needed:** bail with "No image generation needed for this task." Do not write any files.
- **No reference images found:** proceed; phase 2 prompts use stronger textual style description.
- **`.gpt-manual-gen/prompts.md` already exists:** ask resume vs. start-fresh before any other action.
- **`.gitignore` missing:** create it with `.gpt-manual-gen/` as the only line.
- **Project has no asset folders:** no reference scan possible from those roots; only the target output directory's siblings count.
- **User regenerates only some images:** the next Phase 4 run only re-verifies `pending`/`failed:` entries; previously-verified ones stay untouched.
- **GPT Image 2 cannot match a prompt after retries:** offer to refine the prompt (more specific descriptors, swap reference, adjust constraints) before another regeneration attempt.

## Out of scope

- Calling GPT Image 2 via API. This skill is explicitly manual.
- Resizing images to non-native dimensions. The project's build pipeline handles that.
- Generating SVGs. GPT Image 2 outputs raster only.
- Multiple unrelated batches in one invocation. One cohesive batch per run.
