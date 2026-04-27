# gpt-manual-gen plugin — design

## Purpose

A Claude Code plugin that orchestrates a manual image-generation workflow: while building a UI, Claude identifies needed images, drafts descriptive GPT Image 2 prompts (with reference images for visual consistency), hands the list off to the user to run in ChatGPT manually, then verifies the saved images both programmatically and visually before continuing.

The plugin is "manual" because GPT Image 2 isn't called via API — the user copies prompts into ChatGPT, downloads the results, and saves them to disk. Claude handles everything around that human-in-the-loop step.

## Trigger

The plugin exposes both a slash command and an auto-triggering skill:

- **`/gpt-manual-gen`** — explicit entry point when the user wants to plan images now.
- **Skill auto-trigger** — the skill's description matches when Claude is mid-build and recognizes images are needed (e.g., "an icon for the search button", "a hero illustration", "a placeholder avatar"). This catches cases where the user didn't think to invoke a command.

Both paths run the same workflow body.

## Plugin layout

```
gpt-manual-gen/
├── .claude-plugin/
│   └── plugin.json                # plugin manifest
├── commands/
│   └── gpt-manual-gen.md          # slash command — invokes the skill
├── skills/
│   └── gpt-manual-gen/
│       └── SKILL.md               # workflow body, auto-trigger description
└── scripts/
    └── verify-images.mjs          # programmatic image checks
```

## Workflow phases

### 1. Survey

Claude identifies what images the current task needs (count, purpose, target paths) and scans the project for reference images:

- **Search roots:** `public/`, `assets/`, `src/assets/`, `static/`, plus the target output directory of each new image.
- **Extensions:** `.png`, `.jpg`, `.jpeg`, `.webp`, `.svg`.
- **Style extraction:** for any references found, Claude opens them with `Read` (multimodal) and extracts visual identity cues: palette, stroke weight, corner radius, illustration style, lighting, subject framing.

If no references are found, Claude proceeds — phase 2 prompts will lean harder on textual style description.

### 2. Plan

Claude writes `.gpt-manual-gen/prompts.md` at the project root, with one entry per image:

```markdown
# GPT image prompts

Generated 2026-04-27 — generate each image in ChatGPT (GPT Image 2),
save to the listed path, then say "continue" to verify.

## 1. assets/icons/search.png
- **Format:** PNG (transparent background)
- **Native size:** 1024×1024
- **Reference to attach:** assets/icons/home.png
- **Status:** pending
- **Prompt:**
  > A minimal search icon: magnifying glass with handle at lower-right,
  > 2px uniform stroke weight, rounded line caps and joins, monochrome
  > teal (#0EA5E9) on transparent background, centered with ~120px
  > padding from edges, geometric and balanced — matching the
  > attached reference's stroke style and proportions exactly.
```

Per-entry fields:

| Field | Notes |
|---|---|
| Heading | `## N. <target path>` — N is sequential, path is where the user saves. |
| Format | `PNG (transparent background)`, `PNG`, or `JPG`. Drives the format check in verification. |
| Native size | One of GPT Image 2's native output sizes (1024×1024, 1792×1024, 1024×1792). Resizing for final use is the project build pipeline's job, not this plugin's. |
| Reference to attach | Optional — path to project image the user uploads alongside the prompt in ChatGPT. Omitted (line absent) if no reference applies. |
| Status | `pending` initially. Updated by Claude to `verified` after the entry passes both programmatic and visual checks, or `failed: <reason>` after a failure. Drives partial-retry logic. |
| Prompt | Block-quoted descriptive prompt. Should be specific about subject, composition, palette, style, and reference-matching where applicable. |

Claude also writes a short summary to chat: `"N images planned in .gpt-manual-gen/prompts.md — generate them in ChatGPT, then say 'continue' when done."` Then stops.

### 3. Handoff

Claude waits. The user opens ChatGPT, attaches references where indicated, pastes prompts, downloads images, saves to the listed paths.

### 4. Verify

When the user signals continuation ("continue", "done", "ready", or similar), Claude only verifies entries whose `Status:` is `pending` or `failed: ...` — entries already marked `verified` are skipped entirely (no script run, no visual read, no token cost).

For each entry that needs verification:

1. **Programmatic check:** runs `scripts/verify-images.mjs <prompts.md>`. The script parses the file, filters to entries needing verification (Status not `verified`), then for each:
   - File exists at the listed path.
   - File is a valid image and the format matches the requested Format field (PNG, transparent PNG, or JPG — anything else, including WebP, fails).
   - Image dimensions match the requested native size exactly.
   - File size is plausible (≥ 10 KB, ≤ 10 MB) — a sanity check, not a quality measure.

   The script writes results back into `prompts.md`, updating each entry's Status to `verified` (passed all programmatic checks) or `failed: <reason>` (e.g. `failed: file not found`, `failed: dimensions 1024×768 expected 1024×1024`). Entries marked `failed` here do not advance to step 2 — there's no point visually inspecting a missing or malformed image.

2. **Visual check:** Claude opens each entry that the script marked `verified` (passed programmatic check) and visually inspects with `Read`:
   - Subject matches the prompt.
   - Style matches the reference (when one was attached).
   - No glaring artifacts (extra elements, wrong color, distorted geometry).

   On visual mismatch, Claude updates that entry's Status from `verified` back to `failed: <reason>`. Otherwise it stays `verified`.

3. **Report:** Claude consolidates findings:
   - **All `verified`:** announce success, proceed to phase 5.
   - **Some `failed`:** list specific failing entries with diagnosis. User regenerates only those (or asks Claude to refine the prompt), then signals continuation again. Loop repeats — only `pending`/`failed` entries are reprocessed.

### 5. Cleanup

Once every entry passes both checks, Claude deletes `.gpt-manual-gen/` (the artifacts are now the images themselves at their final paths) and resumes the original UI work.

## Edge cases

- **Stale state on entry:** if `.gpt-manual-gen/prompts.md` already exists when the workflow starts (interrupted previous run), Claude reads it and asks the user: "resume from this previous run, or start fresh?" — does not silently overwrite.
- **No reference images found:** workflow proceeds, prompts lean on textual style description.
- **No images actually needed:** skill bails early with `"no image generation needed for this task"` — does not write a file or block the user.
- **Partial regeneration:** if N of M images failed and the user regenerates them, the next "continue" only re-verifies the previously-failing ones.
- **`.gitignore` first run:** Claude appends `.gpt-manual-gen/` to the project's `.gitignore` (creating the file if missing) on first run. This protects against partial state being committed if a workflow is interrupted.
- **Project has no obvious asset folder:** survey still runs against the listed roots (no-op if they don't exist) and the target output directory's parent.

## Out of scope

- Calling GPT Image 2 via API (this plugin is explicitly manual).
- Resizing/cropping images to non-native dimensions — handled by the project's build pipeline.
- Generating SVG (GPT Image 2 outputs raster).
- Batch generation across multiple unrelated UI tasks in one invocation — the skill handles one cohesive batch per run.
