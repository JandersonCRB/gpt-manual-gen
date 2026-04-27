# gpt-manual-gen

A Claude Code plugin for a human-in-the-loop image-generation workflow using ChatGPT's GPT Image 2.

## What it does

When you're building a UI and need images (icons, illustrations, hero shots, avatars), this plugin makes Claude:

1. **Survey** the UI work to identify what images are needed, and scan the project for existing reference images so generated images match your visual identity.
2. **Plan** by writing `.gpt-manual-gen/prompts.md` — a checklist of target paths + descriptive prompts + reference attachments for ChatGPT.
3. **Hand off** to you — open ChatGPT, run each prompt (with reference image attached where indicated), save the result to the listed path.
4. **Verify** when you say "continue" — runs file/format/dimension checks via a Node script, then visually inspects each image and reports any mismatches for regeneration.
5. **Clean up** the working directory and resume the original UI work once everything checks out.

## Triggers

- `/gpt-manual-gen` — explicit invocation
- Auto-triggers when Claude detects images are needed during UI work

## Requirements

- Claude Code
- Node.js 18+
- A ChatGPT account with GPT Image 2 access

## Install

```
/plugin install JandersonCRB/gpt-manual-gen
```

## Design

See [docs/superpowers/specs/2026-04-27-gpt-manual-gen-plugin-design.md](docs/superpowers/specs/2026-04-27-gpt-manual-gen-plugin-design.md).
