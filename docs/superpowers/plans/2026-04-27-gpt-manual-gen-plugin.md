# gpt-manual-gen Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that drives a manual image-generation workflow — Claude drafts GPT Image 2 prompts with reference-image awareness, the user runs them in ChatGPT, then a Node verification script + Claude's visual review confirm the saved images before development continues.

**Architecture:** A standard Claude Code plugin with two entry points (slash command + auto-triggering skill) sharing one workflow body. A small Node script (`scripts/verify-images.mjs`) handles programmatic verification: parses the `prompts.md` plan file, runs file/format/dimension checks on each entry, and writes results back into the file as `Status:` updates. Claude consumes those Status updates and adds a visual review pass.

**Tech Stack:**
- Plugin: standard Claude Code plugin layout (`.claude-plugin/plugin.json`, `commands/`, `skills/`)
- Script runtime: Node.js 18+ (uses built-in `node:test`, ESM via `.mjs`)
- One runtime dependency: `image-size` (small, no native deps, reads dimensions from image headers)
- No build step

**Repo:** https://github.com/JandersonCRB/gpt-manual-gen (already initialized, public, main branch)

---

## File Structure

```
gpt-manual-gen/
├── .claude-plugin/
│   └── plugin.json                    # plugin manifest
├── commands/
│   └── gpt-manual-gen.md              # /gpt-manual-gen slash command
├── skills/
│   └── gpt-manual-gen/
│       └── SKILL.md                   # auto-trigger skill (the workflow body)
├── scripts/
│   ├── verify-images.mjs              # CLI entry point
│   └── lib/
│       ├── parse-prompts.mjs          # parses prompts.md into structured entries
│       ├── verify-entry.mjs           # per-entry programmatic check
│       └── write-status.mjs           # rewrites Status fields in prompts.md
├── tests/
│   ├── parse-prompts.test.mjs
│   ├── verify-entry.test.mjs
│   └── write-status.test.mjs
├── package.json
├── README.md
├── .gitignore                         # already exists
└── docs/superpowers/                  # spec + plan (already exist)
```

**Boundaries:**
- `parse-prompts.mjs` — pure: string in, structured array out. No I/O.
- `verify-entry.mjs` — accepts injected `fs` and `getDimensions` for testability. No `prompts.md` knowledge.
- `write-status.mjs` — pure: original content + status updates → new content. No I/O.
- `verify-images.mjs` — orchestrator: reads file, calls parse → verify → write, exits. The only file with side effects.

---

## Task 1: Project scaffolding and plugin manifest

**Files:**
- Create: `package.json`
- Create: `.claude-plugin/plugin.json`
- Create: `README.md`

This task creates the project skeleton — no logic, no tests yet.

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "gpt-manual-gen",
  "version": "0.1.0",
  "description": "Claude Code plugin for manual image generation via GPT Image 2",
  "type": "module",
  "private": true,
  "scripts": {
    "test": "node --test tests/"
  },
  "dependencies": {
    "image-size": "^1.2.0"
  },
  "engines": {
    "node": ">=18"
  }
}
```

- [ ] **Step 2: Install dependencies**

Run: `npm install`
Expected: creates `node_modules/` and `package-lock.json`. No errors.

- [ ] **Step 3: Create `.claude-plugin/plugin.json`**

```json
{
  "name": "gpt-manual-gen",
  "version": "0.1.0",
  "description": "Manual image generation workflow via GPT Image 2 — Claude drafts prompts, user runs in ChatGPT, Claude verifies"
}
```

- [ ] **Step 4: Create `README.md`**

```markdown
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

\`\`\`
/plugin install JandersonCRB/gpt-manual-gen
\`\`\`

## Design

See [docs/superpowers/specs/2026-04-27-gpt-manual-gen-plugin-design.md](docs/superpowers/specs/2026-04-27-gpt-manual-gen-plugin-design.md).
```

- [ ] **Step 5: Verify the structure**

Run: `ls -la .claude-plugin/ && ls -la && cat package.json`
Expected: shows `plugin.json` in `.claude-plugin/`, shows `package.json` and `README.md` in root, and the `package.json` content matches step 1.

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json .claude-plugin/plugin.json README.md
git commit -m "Add plugin manifest and Node project scaffolding"
```

---

## Task 2: prompts.md parser (TDD)

**Files:**
- Create: `tests/parse-prompts.test.mjs`
- Create: `scripts/lib/parse-prompts.mjs`

The parser converts a `prompts.md` file's text content into an array of structured entries. Pure function — no I/O. The CLI entry point will read the file and pass the string in.

**Output shape per entry:**
```js
{
  index: 1,                                // sequential number from heading
  path: 'assets/icons/search.png',         // target path relative to project root
  format: 'PNG (transparent background)',  // raw Format field value
  nativeSize: '1024×1024',                 // raw Native size field value
  reference: 'assets/icons/home.png',      // raw Reference field value, or null
  status: 'pending',                       // raw Status field value
  prompt: 'A minimal search icon: ...'     // joined prompt text from > block
}
```

- [ ] **Step 1: Write the failing test**

Create `tests/parse-prompts.test.mjs`:

```javascript
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { parsePromptsFile } from '../scripts/lib/parse-prompts.mjs';

const SAMPLE = `# GPT image prompts

Generated 2026-04-27 — generate each image in ChatGPT (GPT Image 2),
save to the listed path, then say "continue" to verify.

## 1. assets/icons/search.png
- **Format:** PNG (transparent background)
- **Native size:** 1024×1024
- **Reference to attach:** assets/icons/home.png
- **Status:** pending
- **Prompt:**
  > A minimal search icon: magnifying glass with handle at lower-right,
  > 2px uniform stroke weight, monochrome teal on transparent background.

## 2. assets/hero.jpg
- **Format:** JPG
- **Native size:** 1792×1024
- **Status:** verified
- **Prompt:**
  > Wide hero illustration of an open notebook on a wooden desk,
  > soft morning light from the left, muted earth tones.

## 3. assets/icons/menu.png
- **Format:** PNG (transparent background)
- **Native size:** 1024×1024
- **Reference to attach:** assets/icons/home.png
- **Status:** failed: dimensions 1024×768 expected 1024×1024
- **Prompt:**
  > A minimal hamburger menu icon, 2px stroke, monochrome teal.
`;

test('parses three entries with all fields', () => {
  const entries = parsePromptsFile(SAMPLE);
  assert.equal(entries.length, 3);
});

test('first entry has all fields populated correctly', () => {
  const [e1] = parsePromptsFile(SAMPLE);
  assert.equal(e1.index, 1);
  assert.equal(e1.path, 'assets/icons/search.png');
  assert.equal(e1.format, 'PNG (transparent background)');
  assert.equal(e1.nativeSize, '1024×1024');
  assert.equal(e1.reference, 'assets/icons/home.png');
  assert.equal(e1.status, 'pending');
  assert.match(e1.prompt, /minimal search icon/);
  assert.match(e1.prompt, /monochrome teal on transparent background\.$/);
});

test('second entry has no reference (field absent)', () => {
  const [, e2] = parsePromptsFile(SAMPLE);
  assert.equal(e2.reference, null);
  assert.equal(e2.format, 'JPG');
  assert.equal(e2.status, 'verified');
});

test('failed status preserves the reason', () => {
  const [, , e3] = parsePromptsFile(SAMPLE);
  assert.equal(e3.status, 'failed: dimensions 1024×768 expected 1024×1024');
});

test('prompt joins multi-line block quote with newlines preserved', () => {
  const [e1] = parsePromptsFile(SAMPLE);
  const lines = e1.prompt.split('\n');
  assert.equal(lines.length, 2);
  assert.match(lines[0], /minimal search icon/);
  assert.match(lines[1], /2px uniform stroke/);
});

test('returns empty array when no entries', () => {
  const entries = parsePromptsFile('# GPT image prompts\n\nNothing here.\n');
  assert.deepEqual(entries, []);
});

test('throws on invalid heading format', () => {
  const bad = '## not-a-numbered-heading\n- **Format:** PNG\n';
  assert.throws(() => parsePromptsFile(bad), /heading/i);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: tests fail with `Cannot find module '../scripts/lib/parse-prompts.mjs'` or similar import error.

- [ ] **Step 3: Implement the parser**

Create `scripts/lib/parse-prompts.mjs`:

```javascript
const HEADING_RE = /^## (\d+)\.\s+(.+?)\s*$/;
const FIELD_RE = /^- \*\*([^:]+):\*\*\s*(.*)$/;
const QUOTE_RE = /^\s*>\s?(.*)$/;
const BAD_HEADING_RE = /^## (?!\d)/;

const FIELD_KEY_MAP = {
  'format': 'format',
  'native size': 'nativeSize',
  'reference to attach': 'reference',
  'status': 'status',
};

export function parsePromptsFile(content) {
  const lines = content.split('\n');
  const entries = [];
  let current = null;

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];

    if (BAD_HEADING_RE.test(line)) {
      throw new Error(`Invalid section heading (must be "## N. <path>"): ${line}`);
    }

    const headingMatch = line.match(HEADING_RE);
    if (headingMatch) {
      if (current) entries.push(current);
      current = {
        index: Number(headingMatch[1]),
        path: headingMatch[2].trim(),
        format: null,
        nativeSize: null,
        reference: null,
        status: null,
        prompt: null,
      };
      continue;
    }

    if (!current) continue;

    const fieldMatch = line.match(FIELD_RE);
    if (!fieldMatch) continue;

    const rawKey = fieldMatch[1].trim().toLowerCase();
    const value = fieldMatch[2].trim();

    if (rawKey === 'prompt') {
      const promptLines = [];
      let j = i + 1;
      while (j < lines.length) {
        const next = lines[j];
        if (HEADING_RE.test(next)) break;
        const qm = next.match(QUOTE_RE);
        if (qm) {
          promptLines.push(qm[1]);
        } else if (next.trim() === '' && promptLines.length === 0) {
          // skip leading blank between "- **Prompt:**" and first quote line
        } else {
          break;
        }
        j++;
      }
      current.prompt = promptLines.join('\n').trim();
      i = j - 1;
      continue;
    }

    const mappedKey = FIELD_KEY_MAP[rawKey];
    if (mappedKey) current[mappedKey] = value;
  }

  if (current) entries.push(current);
  return entries;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test`
Expected: all 7 tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/parse-prompts.mjs tests/parse-prompts.test.mjs
git commit -m "Add prompts.md parser with structured entry output"
```

---

## Task 3: Status writer (TDD)

**Files:**
- Create: `tests/write-status.test.mjs`
- Create: `scripts/lib/write-status.mjs`

The writer takes original `prompts.md` content + a map of `index → newStatus` and returns updated content with `Status:` fields rewritten in place. Pure function. Preserves all other formatting (whitespace, prompts, etc.) exactly.

- [ ] **Step 1: Write the failing test**

Create `tests/write-status.test.mjs`:

```javascript
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { writeStatusUpdates } from '../scripts/lib/write-status.mjs';

const SAMPLE = `# GPT image prompts

## 1. assets/icons/search.png
- **Format:** PNG (transparent background)
- **Native size:** 1024×1024
- **Status:** pending
- **Prompt:**
  > Search icon prompt.

## 2. assets/hero.jpg
- **Format:** JPG
- **Native size:** 1792×1024
- **Status:** pending
- **Prompt:**
  > Hero prompt.
`;

test('updates a single entry status', () => {
  const updates = new Map([[1, 'verified']]);
  const result = writeStatusUpdates(SAMPLE, updates);
  assert.match(result, /## 1\. assets\/icons\/search\.png[\s\S]*\*\*Status:\*\* verified/);
  assert.match(result, /## 2\. assets\/hero\.jpg[\s\S]*\*\*Status:\*\* pending/);
});

test('updates multiple entries independently', () => {
  const updates = new Map([
    [1, 'verified'],
    [2, 'failed: file not found'],
  ]);
  const result = writeStatusUpdates(SAMPLE, updates);
  assert.match(result, /## 1\.[\s\S]*\*\*Status:\*\* verified/);
  assert.match(result, /## 2\.[\s\S]*\*\*Status:\*\* failed: file not found/);
});

test('preserves prompt content unchanged', () => {
  const updates = new Map([[1, 'verified']]);
  const result = writeStatusUpdates(SAMPLE, updates);
  assert.match(result, /> Search icon prompt\./);
  assert.match(result, /> Hero prompt\./);
});

test('preserves heading and surrounding structure', () => {
  const updates = new Map([[1, 'verified']]);
  const result = writeStatusUpdates(SAMPLE, updates);
  assert.match(result, /^# GPT image prompts/);
});

test('no-op when index not present', () => {
  const updates = new Map([[99, 'verified']]);
  const result = writeStatusUpdates(SAMPLE, updates);
  assert.equal(result, SAMPLE);
});

test('overwrites a previous failed status', () => {
  const withFailed = SAMPLE.replace('**Status:** pending', '**Status:** failed: file not found');
  const updates = new Map([[1, 'verified']]);
  const result = writeStatusUpdates(withFailed, updates);
  assert.match(result, /## 1\.[\s\S]*\*\*Status:\*\* verified/);
  assert.doesNotMatch(result, /failed: file not found/);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: write-status tests fail with module-not-found; parse-prompts tests still pass.

- [ ] **Step 3: Implement the writer**

Create `scripts/lib/write-status.mjs`:

```javascript
export function writeStatusUpdates(content, updates) {
  let result = content;
  for (const [index, newStatus] of updates) {
    const re = new RegExp(
      `(^## ${index}\\.[^\\n]*\\n(?:[^\\n]*\\n)*?- \\*\\*Status:\\*\\* )([^\\n]*)`,
      'm',
    );
    result = result.replace(re, (_, prefix) => `${prefix}${newStatus}`);
  }
  return result;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test`
Expected: all tests pass (parse-prompts: 7, write-status: 6).

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/write-status.mjs tests/write-status.test.mjs
git commit -m "Add status writer that rewrites prompts.md Status fields"
```

---

## Task 4: Per-entry verification logic (TDD)

**Files:**
- Create: `tests/verify-entry.test.mjs`
- Create: `scripts/lib/verify-entry.mjs`

The verifier checks one parsed entry against the file system. Returns `{ ok: true }` or `{ ok: false, reason: '...' }`. `fs` and `getDimensions` are injected so tests don't need real image files.

**Checks (in order, short-circuit on first failure):**
1. File exists at `entry.path` (resolved against project root).
2. File size in `[10 KB, 10 MB]`.
3. File extension matches Format field (`.png` for PNG variants, `.jpg`/`.jpeg` for JPG).
4. Image dimensions parsed via `getDimensions` match `entry.nativeSize`.

- [ ] **Step 1: Write the failing test**

Create `tests/verify-entry.test.mjs`:

```javascript
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { verifyEntry } from '../scripts/lib/verify-entry.mjs';

const PNG_ENTRY = {
  index: 1,
  path: 'assets/icons/search.png',
  format: 'PNG (transparent background)',
  nativeSize: '1024×1024',
  reference: null,
  status: 'pending',
  prompt: '...',
};

const JPG_ENTRY = {
  ...PNG_ENTRY,
  index: 2,
  path: 'assets/hero.jpg',
  format: 'JPG',
  nativeSize: '1792×1024',
};

function makeFs({ exists = true, size = 100_000 } = {}) {
  return {
    existsSync: () => exists,
    statSync: () => ({ size }),
  };
}

function makeDims(width, height) {
  return () => ({ width, height });
}

test('passes when all checks succeed (PNG)', () => {
  const result = verifyEntry(PNG_ENTRY, '/proj', {
    fs: makeFs(),
    getDimensions: makeDims(1024, 1024),
  });
  assert.deepEqual(result, { ok: true });
});

test('passes when all checks succeed (JPG)', () => {
  const result = verifyEntry(JPG_ENTRY, '/proj', {
    fs: makeFs(),
    getDimensions: makeDims(1792, 1024),
  });
  assert.deepEqual(result, { ok: true });
});

test('fails when file does not exist', () => {
  const result = verifyEntry(PNG_ENTRY, '/proj', {
    fs: makeFs({ exists: false }),
    getDimensions: makeDims(1024, 1024),
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /file not found/);
});

test('fails when file too small', () => {
  const result = verifyEntry(PNG_ENTRY, '/proj', {
    fs: makeFs({ size: 1000 }),
    getDimensions: makeDims(1024, 1024),
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /too small/);
});

test('fails when file too large', () => {
  const result = verifyEntry(PNG_ENTRY, '/proj', {
    fs: makeFs({ size: 20_000_000 }),
    getDimensions: makeDims(1024, 1024),
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /too large/);
});

test('fails when extension does not match PNG format', () => {
  const entry = { ...PNG_ENTRY, path: 'assets/icons/search.webp' };
  const result = verifyEntry(entry, '/proj', {
    fs: makeFs(),
    getDimensions: makeDims(1024, 1024),
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /extension/i);
  assert.match(result.reason, /\.webp/);
});

test('fails when extension does not match JPG format', () => {
  const entry = { ...JPG_ENTRY, path: 'assets/hero.png' };
  const result = verifyEntry(entry, '/proj', {
    fs: makeFs(),
    getDimensions: makeDims(1792, 1024),
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /extension/i);
});

test('accepts .jpeg extension for JPG format', () => {
  const entry = { ...JPG_ENTRY, path: 'assets/hero.jpeg' };
  const result = verifyEntry(entry, '/proj', {
    fs: makeFs(),
    getDimensions: makeDims(1792, 1024),
  });
  assert.deepEqual(result, { ok: true });
});

test('fails when dimensions mismatch', () => {
  const result = verifyEntry(PNG_ENTRY, '/proj', {
    fs: makeFs(),
    getDimensions: makeDims(1024, 768),
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /1024×768/);
  assert.match(result.reason, /1024×1024/);
});

test('fails when getDimensions returns null', () => {
  const result = verifyEntry(PNG_ENTRY, '/proj', {
    fs: makeFs(),
    getDimensions: () => null,
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /dimensions/i);
});

test('fails on unrecognized format field', () => {
  const entry = { ...PNG_ENTRY, format: 'TIFF' };
  const result = verifyEntry(entry, '/proj', {
    fs: makeFs(),
    getDimensions: makeDims(1024, 1024),
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /format/i);
});

test('fails on unparseable native size', () => {
  const entry = { ...PNG_ENTRY, nativeSize: 'not-a-size' };
  const result = verifyEntry(entry, '/proj', {
    fs: makeFs(),
    getDimensions: makeDims(1024, 1024),
  });
  assert.equal(result.ok, false);
  assert.match(result.reason, /native size/i);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: verify-entry tests fail with module-not-found; other tests still pass.

- [ ] **Step 3: Implement the verifier**

Create `scripts/lib/verify-entry.mjs`:

```javascript
import { resolve, extname } from 'node:path';

const MIN_FILE_SIZE = 10 * 1024;
const MAX_FILE_SIZE = 10 * 1024 * 1024;

const PNG_EXTS = new Set(['.png']);
const JPG_EXTS = new Set(['.jpg', '.jpeg']);

export function verifyEntry(entry, projectRoot, deps) {
  const { fs, getDimensions } = deps;
  const fullPath = resolve(projectRoot, entry.path);

  if (!fs.existsSync(fullPath)) {
    return { ok: false, reason: 'file not found' };
  }

  const { size } = fs.statSync(fullPath);
  if (size < MIN_FILE_SIZE) {
    return { ok: false, reason: `file too small (${size} bytes, minimum ${MIN_FILE_SIZE})` };
  }
  if (size > MAX_FILE_SIZE) {
    return { ok: false, reason: `file too large (${size} bytes, maximum ${MAX_FILE_SIZE})` };
  }

  const formatLower = entry.format.toLowerCase();
  const ext = extname(entry.path).toLowerCase();
  let allowedExts;
  if (formatLower.includes('png')) {
    allowedExts = PNG_EXTS;
  } else if (formatLower.includes('jpg') || formatLower.includes('jpeg')) {
    allowedExts = JPG_EXTS;
  } else {
    return { ok: false, reason: `unrecognized format field: "${entry.format}"` };
  }
  if (!allowedExts.has(ext)) {
    return { ok: false, reason: `extension ${ext} does not match format "${entry.format}"` };
  }

  const expected = parseSize(entry.nativeSize);
  if (!expected) {
    return { ok: false, reason: `unparseable native size: "${entry.nativeSize}"` };
  }

  const actual = getDimensions(fullPath);
  if (!actual || typeof actual.width !== 'number' || typeof actual.height !== 'number') {
    return { ok: false, reason: 'unable to read image dimensions' };
  }

  if (actual.width !== expected.width || actual.height !== expected.height) {
    return {
      ok: false,
      reason: `dimensions ${actual.width}×${actual.height} expected ${expected.width}×${expected.height}`,
    };
  }

  return { ok: true };
}

function parseSize(value) {
  const m = value.match(/(\d+)\s*[×x]\s*(\d+)/);
  if (!m) return null;
  return { width: Number(m[1]), height: Number(m[2]) };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test`
Expected: all tests pass (parse-prompts: 7, write-status: 6, verify-entry: 12).

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/verify-entry.mjs tests/verify-entry.test.mjs
git commit -m "Add per-entry programmatic image verification"
```

---

## Task 5: CLI entry point (`verify-images.mjs`)

**Files:**
- Create: `scripts/verify-images.mjs`

The CLI orchestrates the three library functions. Reads `prompts.md`, parses, verifies each entry whose status is not `verified`, writes updates back, prints a summary, exits 0 if all pass / 1 otherwise.

**Usage:** `node scripts/verify-images.mjs <path-to-prompts.md>`

The script resolves image paths relative to the directory containing the prompts file (which is the project root, since the spec puts `prompts.md` at `.gpt-manual-gen/prompts.md` — so paths resolve relative to the parent of `.gpt-manual-gen/`, i.e. the project root).

- [ ] **Step 1: Implement the CLI**

Create `scripts/verify-images.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync, writeFileSync, existsSync, statSync } from 'node:fs';
import { resolve, dirname } from 'node:path';
import { imageSize } from 'image-size';
import { parsePromptsFile } from './lib/parse-prompts.mjs';
import { verifyEntry } from './lib/verify-entry.mjs';
import { writeStatusUpdates } from './lib/write-status.mjs';

const NEEDS_VERIFICATION = (status) => status === 'pending' || status?.startsWith('failed');

function main() {
  const promptsPath = process.argv[2];
  if (!promptsPath) {
    console.error('Usage: node scripts/verify-images.mjs <path-to-prompts.md>');
    process.exit(2);
  }

  if (!existsSync(promptsPath)) {
    console.error(`prompts file not found: ${promptsPath}`);
    process.exit(2);
  }

  const content = readFileSync(promptsPath, 'utf8');
  const entries = parsePromptsFile(content);

  // Project root = parent of the directory containing prompts.md.
  // Spec: prompts.md lives at <projectRoot>/.gpt-manual-gen/prompts.md
  const projectRoot = resolve(dirname(promptsPath), '..');

  const fs = {
    existsSync,
    statSync,
  };
  const getDimensions = (path) => {
    try {
      const result = imageSize(path);
      return result ? { width: result.width, height: result.height } : null;
    } catch {
      return null;
    }
  };

  const updates = new Map();
  let toVerify = 0;
  let passed = 0;
  let failed = 0;
  let skipped = 0;

  for (const entry of entries) {
    if (!NEEDS_VERIFICATION(entry.status)) {
      skipped++;
      continue;
    }
    toVerify++;
    const result = verifyEntry(entry, projectRoot, { fs, getDimensions });
    if (result.ok) {
      updates.set(entry.index, 'verified');
      passed++;
      console.log(`  [PASS] ${entry.index}. ${entry.path}`);
    } else {
      updates.set(entry.index, `failed: ${result.reason}`);
      failed++;
      console.log(`  [FAIL] ${entry.index}. ${entry.path} — ${result.reason}`);
    }
  }

  if (updates.size > 0) {
    const updated = writeStatusUpdates(content, updates);
    writeFileSync(promptsPath, updated, 'utf8');
  }

  console.log('');
  console.log(`Verified: ${passed}/${toVerify} passed, ${failed} failed, ${skipped} already verified.`);

  process.exit(failed === 0 ? 0 : 1);
}

main();
```

- [ ] **Step 2: Smoke test against a real PNG**

Create a temporary fixture and run the script end-to-end. This is a manual smoke test, not an automated one (real image files are awkward in CI).

```bash
mkdir -p /tmp/gmg-smoke/.gpt-manual-gen /tmp/gmg-smoke/assets
# Generate a 1024x1024 PNG using a tiny inline node script
node -e "
const { writeFileSync } = require('fs');
const size = 1024;
// A 1024x1024 solid gray PNG. We use sharp-free approach: write a minimal but valid PNG.
// Easier: download/copy any 1024x1024 PNG. For smoke, we'll skip and instead use a known image.
console.log('Place a real 1024x1024 PNG at /tmp/gmg-smoke/assets/test.png and re-run.');
"
```

Skip generating fixtures here. Instead, do a real smoke test by hand: drop any 1024×1024 PNG into `/tmp/gmg-smoke/assets/test.png`, then write a sample `prompts.md`:

```bash
cat > /tmp/gmg-smoke/.gpt-manual-gen/prompts.md <<'EOF'
# GPT image prompts

## 1. assets/test.png
- **Format:** PNG (transparent background)
- **Native size:** 1024×1024
- **Status:** pending
- **Prompt:**
  > Test prompt.
EOF
node scripts/verify-images.mjs /tmp/gmg-smoke/.gpt-manual-gen/prompts.md
```

Expected output (if a 1024×1024 PNG was placed):
```
  [PASS] 1. assets/test.png

Verified: 1/1 passed, 0 failed, 0 already verified.
```

If no real image is available, skip this smoke step — the unit tests already cover the logic.

- [ ] **Step 3: Verify error handling for missing argument**

Run: `node scripts/verify-images.mjs`
Expected: prints `Usage: node scripts/verify-images.mjs <path-to-prompts.md>`, exits with code 2.

Run: `echo $?` (Unix) or `$LASTEXITCODE` (PowerShell)
Expected: 2.

- [ ] **Step 4: Verify error handling for missing file**

Run: `node scripts/verify-images.mjs /nonexistent/prompts.md`
Expected: prints `prompts file not found: /nonexistent/prompts.md`, exits with code 2.

- [ ] **Step 5: Commit**

```bash
git add scripts/verify-images.mjs
git commit -m "Add verify-images CLI orchestrating parse, verify, and write"
```

---

## Task 6: Skill body (`SKILL.md`)

**Files:**
- Create: `skills/gpt-manual-gen/SKILL.md`

This is the workflow body Claude follows. It must:
- Have frontmatter with `name` (matching the directory) and a `description` that triggers auto-activation when image needs are detected.
- Walk through the 5 phases from the spec.
- Specify exactly how `prompts.md` is formatted (so Claude generates it correctly).
- Cover all edge cases from the spec.

- [ ] **Step 1: Create the skill file**

Create `skills/gpt-manual-gen/SKILL.md`:

`````markdown
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

If activated and no images are actually needed for the current task, bail early with: "No image generation needed for this task." Do not write any files.

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

Wait for the answer. If resume: skip ahead to Phase 4. If start fresh: delete `.gpt-manual-gen/` and continue below.

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

Run the verification script:

```
node <plugin path>/scripts/verify-images.mjs .gpt-manual-gen/prompts.md
```

The plugin path resolution is environment-dependent. If running from inside the plugin's repo, use `scripts/verify-images.mjs`. If installed as a Claude Code plugin, the path is `${CLAUDE_PLUGIN_ROOT}/scripts/verify-images.mjs`.

The script:
1. Parses `prompts.md`.
2. For each entry whose Status is `pending` or `failed: ...`, runs file-existence, format-extension, dimension, and file-size checks.
3. Writes results back into `prompts.md` as updated `Status:` lines (`verified` or `failed: <reason>`).
4. Prints a per-entry pass/fail summary and exits 0 (all passed) or 1 (some failed).

Read the updated `prompts.md` after the script runs. Note which entries are now `verified` and which are `failed: ...`.

### Step 4b: Visual check

For every entry now marked `verified` (and only those — entries the script marked `failed` skip this step), open the image with `Read` and judge:

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
`````

- [ ] **Step 2: Verify the file is well-formed**

Run: `head -5 skills/gpt-manual-gen/SKILL.md`
Expected: shows the YAML frontmatter (`---`, `name:`, `description:`, `---`) followed by the H1 heading.

- [ ] **Step 3: Commit**

```bash
git add skills/gpt-manual-gen/SKILL.md
git commit -m "Add gpt-manual-gen skill body with full workflow"
```

---

## Task 7: Slash command

**Files:**
- Create: `commands/gpt-manual-gen.md`

The slash command is a thin entry point that invokes the skill. The user types `/gpt-manual-gen` when they want to plan images explicitly.

- [ ] **Step 1: Create the command file**

Create `commands/gpt-manual-gen.md`:

```markdown
---
description: Plan and verify GPT Image 2 generated images for the current UI work
---

Use the `gpt-manual-gen` skill to drive the manual image-generation workflow for the current task.

If the user has already described which images they need in this conversation, use that as the survey input. Otherwise ask them which UI surface needs images before starting Phase 1.
```

- [ ] **Step 2: Commit**

```bash
git add commands/gpt-manual-gen.md
git commit -m "Add /gpt-manual-gen slash command"
```

---

## Task 8: End-to-end smoke test and final push

**Files:** none — this is verification + push.

- [ ] **Step 1: Run all unit tests**

Run: `npm test`
Expected: all tests pass (parse-prompts: 7, write-status: 6, verify-entry: 12 = 25 total).

- [ ] **Step 2: Verify final tree**

Run: `git ls-files | sort` (works on every platform).

Expected output (committed files only — `node_modules/`, `package-lock.json`'s tracked status varies but should be present, plan & spec already committed):
```
.claude-plugin/plugin.json
.gitignore
README.md
commands/gpt-manual-gen.md
docs/superpowers/plans/2026-04-27-gpt-manual-gen-plugin.md
docs/superpowers/specs/2026-04-27-gpt-manual-gen-plugin-design.md
package-lock.json
package.json
scripts/lib/parse-prompts.mjs
scripts/lib/verify-entry.mjs
scripts/lib/write-status.mjs
scripts/verify-images.mjs
skills/gpt-manual-gen/SKILL.md
tests/parse-prompts.test.mjs
tests/verify-entry.test.mjs
tests/write-status.test.mjs
```

If anything is missing or extra, fix before continuing.

- [ ] **Step 3: Verify the plugin manifest validates**

Run: `node -e "console.log(JSON.parse(require('fs').readFileSync('.claude-plugin/plugin.json', 'utf8')))"`
Expected: prints the parsed manifest object with `name`, `version`, `description`. No errors.

- [ ] **Step 4: Add the implementation plan to git and push**

```bash
git add docs/superpowers/plans/2026-04-27-gpt-manual-gen-plugin.md
git commit -m "Add implementation plan for gpt-manual-gen plugin"
git push origin main
```

Expected: push succeeds; the GitHub repo at https://github.com/JandersonCRB/gpt-manual-gen now reflects all changes.

- [ ] **Step 5: Verify on GitHub**

Run: `gh repo view JandersonCRB/gpt-manual-gen --web` (or open the URL manually).

Confirm: the repo shows the plugin layout, README, spec, and plan.

---

## Out of scope (intentionally)

- Automated tests against real image files. Logic tests inject mocked `getDimensions`; only manual smoke tests use real images.
- Plugin marketplace publication. The repo is push-installable via `/plugin install JandersonCRB/gpt-manual-gen`; further marketplace metadata is left for later.
- A CI workflow. Tests can be added to GitHub Actions later if needed.
- Self-update logic in the verify script. If `prompts.md` formatting evolves, both the SKILL.md format spec and the parser need to change together — that's caught by the parser tests.
