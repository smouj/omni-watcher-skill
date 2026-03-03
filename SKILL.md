---
name: omni-watcher
description: AI-powered file watcher for real-time writing improvement and content generation
version: 2.1.0
author: OpenClaw Team
tags:
  - writing
  - ai
  - automation
  - real-time
  - markdown
  - content
dependencies:
  - node >=18.0.0
  - openai-api
  - chokidar
  - yaml
  - diff
  - ts-node
  - typescript
platforms:
  - linux
  - macos
  - windows
entrypoint: bin/omni-watcher
---

# Omni Watcher

AI-powered real-time file watcher that monitors writing files (Markdown, plain text, JSON, YAML) and provides automatic improvements, completions, and style enforcement as you type.

## Purpose

**Real use cases:**

1. **Documentation writing assistant** - Watch `/docs/` directory and automatically improve Markdown files for clarity, grammar, and technical accuracy while preserving code blocks and frontmatter.

2. **Blog post workflow** - Monitor a `content/posts/` directory. When a `.md` file is saved, analyze sentence structure, suggest better transitions, and ensure SEO-friendly headings.

3. **API documentation guard** - Watch OpenAPI/Swagger YAML files to ensure consistent terminology, detect missing descriptions, and flag undocumented parameters automatically.

4. **Team writing standards enforcement** - Run in CI or development mode to ensure all team members' documentation follows established style guides (AP Style, Google Developer Documentation Style, etc.).

5. **Translation quality monitor** - Watch translated Markdown files to detect awkward phrasing, maintain terminology consistency across language versions, and flag potential mistranslations.

## Scope

**Supported file types:**
- `.md`, `.markdown` (GitHub Flavored Markdown)
- `.txt` (plain text)
- `.yaml`, `.yml` (YAML frontmatter preservation)
- `.json` (JSON with comments)
- `.mdx` (MDX with JSX)

**Active commands:**

```bash
# Start watching with default AI improvements
omni-watcher --watch ./docs

# Watch with specific AI model and custom prompts
omni-watcher --watch ./content --model gpt-4 --prompt "Make tone enthusiastic, add emojis"

# Watch and auto-apply fixes (interactive mode requires --confirm)
omni-watcher --watch ./src/docs --auto-fix --backup-dir ./backups

# Differential mode: show changes without applying
omni-watcher --watch ./notes --preview-only --diff

# Single-file analysis (no watching)
omni-watcher --analyze README.md --output improved.md

# Configure via YAML
omni-watcher --config .omni-watcher.yml

# Watch with custom ignore patterns
omni-watcher --watch . --ignore "**/drafts/**" --ignore "**/*.template.md"

# Integration mode: output to stdout for CI
omni-watcher --ci ./docs --format json --fail-on-quality-score-below 80

# Dry-run with verbose logging
omni-watcher --watch ./docs --dry-run --log-level debug

# Multi-directory watch with different rules per directory
omni-watcher --watch ./api-docs --rules api-rules.yml --watch ./tutorials --rules tutorial-rules.yml
```

## Work Process

**Step-by-step execution:**

1. **Initialization phase**
   ```bash
   # Load config from priority sources:
   # 1. --config file if specified
   # 2. .omni-watcher.yml in current directory
   # 3. .omni-watcher.yml in home directory
   # 4. Default configuration
   ```

2. **File discovery**
   - Recursively scan watched directories
   - Filter by extension (whitelist: md, markdown, txt, yaml, yml, json, mdx)
   - Apply `--ignore` glob patterns
   - Build initial file state with SHA256 hashes

3. **Watcher activation**
   - Use `chokidar` with `awaitWriteFinish: { stabilityThreshold: 500, pollInterval: 100 }`
   - Debounce rapid successive saves (default: 1000ms)
   - Detect file additions, modifications, deletions

4. **Change detection**
   ```typescript
   if (newHash !== oldHash) {
     categorizeChange(file);
     categorizeChange(file);
   }
   ```

5. **AI processing pipeline**
   ```
   Input file → Extract content (preserve frontmatter/code blocks) → Apply custom rules →
   Chunk if >4000 tokens → Send to OpenAI API with system prompt → Receive suggestions →
   Parse diff → Apply changes (interactive or auto) → Generate backup if auto-fix →
   Write improved file → Log summary → Trigger hooks if configured
   ```

6. **Quality scoring**
   - Calculate readability score (Flesch-Kincaid)
   - Check grammar errors (via LanguageTool API or OpenAI)
   - Verify style compliance (custom regex rules)
   - Return exit code 0-100 for CI mode

## Golden Rules

**Specific constraints:**

1. **Frontmatter preservation**: Never modify YAML frontmatter between `---` delimiters in Markdown files. Treat it as immutable unless `--update-frontmatter` flag is explicitly set.

2. **Code block integrity**: All fenced code blocks (```language) must remain unchanged. AI suggestions may only target text outside code blocks. Use regex `^```[\s\S]*?^```$` with multiline flag to extract and skip.

3. **Backup requirement**: When `--auto-fix` is enabled, create timestamped backups in `--backup-dir` before applying changes. Backup filename format: `original-YYYYMMDD-HHMMSS.md`. Retain last 10 backups per file, delete older ones.

4. **Token limit enforcement**: If file content exceeds 4000 tokens, split into logical sections (by headers or paragraphs) and process independently, then merge results. Never truncate content; always process complete sections.

5. **Rate limiting**: Implement exponential backoff for OpenAI API failures. Max retries: 3. On rate limit (429), wait 2^n * 1000ms where n is retry count. Log warnings on retries.

6. **No network**: When `--offline` mode is active, skip AI processing and only validate against local rules. Exit with code 2 to indicate partial processing.

7. **CI compliance**: In `--ci` mode, output must be machine-readable JSON:
   ```json
   {
     "file": "docs/README.md",
     "status": "passed|failed|error",
     "quality_score": 85,
     "issues": ["line 12: passive voice", "line 45: jargon detected"],
     "changes_applied": 3
   }
   ```

8. **Thread safety**: Watcher processes files sequentially, not in parallel, to prevent race conditions when multiple files save simultaneously. Queue size limit: 10 files.

## Examples

**Example 1: Documentation improvement with style guide**

```bash
# Command
omni-watcher --watch ./docs --model gpt-4 --prompt-file .styleguide.txt --auto-fix --backup-dir ./backups

# .styleguide.txt contents:
# Use active voice
# Max sentence length: 25 words
# Avoid passive voice
# Use Oxford comma
# Technical terms: First appearance should be capitalized (API, REST, JSON)

# Input: docs/getting-started.md before
## Introduction
The configuration file is created by the installer. Settings can be modified by the user.

# After auto-fix:
## Introduction
The installer creates the configuration file. You can modify settings.

# Log output:
[INFO] File changed: docs/getting-started.md
[INFO] Processing with GPT-4...
[INFO] Applied 3 improvements (passive voice, Oxford comma)
[INFO] Backup created: backups/getting-started-20240315-143022.md
[INFO] Quality score: 92/100
```

**Example 2: CI mode with quality gate**

```bash
# .github/workflows/docs.yml
- name: Check documentation quality
  run: |
    omni-watcher --ci ./docs --format json --fail-on-quality-score-below 85 > quality-report.json
    cat quality-report.json
    
# Output on failure:
{
  "file": "docs/api.md",
  "status": "failed",
  "quality_score": 78,
  "issues": [
    "line 23: Missing description for 'userId' parameter",
    "line 45: Inconsistent capitalization (api vs API)",
    "line 89: Sentence exceeds 30 words"
  ],
  "changes_applied": 0
}
# Exit code 1 (CI fails)
```

**Example 3: Multi-directory with different rules**

```bash
# Command:
omni-watcher \
  --watch ./api-reference --rules api-rules.yml \
  --watch ./tutorials --rules tutorial-rules.yml \
  --diff --log-level info

# api-rules.yml:
rules:
  - type: terminology
    enforce: ["REST API", "HTTP", "endpoint"]
    forbid: ["rest api", "Rest API", "http"]
  - type: docstring
    require: ["description", "parameters", "returns"]

# When ./api-reference/users.md changes:
[INFO] Applying api-rules.yml to api-reference/users.md
[DIFF] Line 14: 'rest api' → 'REST API'
[DIFF] Line 22: Missing parameter description for 'role'
[PROMPT] Add parameter descriptions for: role, createdAt
```

**Example 4: Single-file analysis for translation quality**

```bash
omni-watcher --analyze ./translations/es/marketing.md --target-language en --check-consistency --output consistency-report.txt

# Output:
Translations consistency check for es/marketing.md
✓ Term consistency: "carrito" always mapped to "cart"
⚠ Potential mismatch: "pago" translated as "payment" (expected: "checkout")
✗ Missing translation: 3 strings found in en version not present in es
Recommendation: Review sections 2.1 and 4.3 for completeness.
```

## Rollback Commands

**Specific rollback procedures:**

```bash
# 1. Revert single file to previous backup (if --backup-dir used)
cp ./backups/getting-started-20240315-143022.md ./docs/getting-started.md

# 2. Restore all files from backup batch (use with caution)
omni-watcher --restore ./backups/batch-20240315-143022 --target ./docs

# 3. Git-based rollback (recommended for git-tracked files)
git checkout HEAD -- docs/changed-file.md
# or for multiple files:
git restore --source=HEAD ./docs/

# 4. Stop watcher gracefully (SIGTERM) or forcefully (SIGKILL)
# Graceful: sends completion signal, finishes current file, exits
pkill -TERM -f "omni-watcher"
# Force:
pkill -KILL -f "omni-watcher"

# 5. Rollback last batch of auto-fixes (if using omni-watcher's ledger)
omni-watcher --undo --since "2024-03-15T14:30:00Z"

# 6. Disable auto-fix mode while preserving watcher
# Edit config or restart without --auto-fix flag:
omni-watcher --watch ./docs  # (no --auto-fix)

# 7. Clear watcher state (new file hashes)
rm -f .omni-watcher-state.json

# 8. Revert configuration to default
mv .omni-watcher.yml .omni-watcher.yml.backup
omni-watcher --watch ./docs  # uses defaults

# Safety check before rolling back:
# 1. List available backups
ls -lht ./backups/
# 2. Verify backup integrity
head -20 ./backups/file-20240315-143022.md
# 3. Confirm file isn't modified since backup
diff ./docs/file.md ./backups/file-20240315-143022.md

# Emergency stop (kill all related processes):
pkill -f "omni-watcher" && echo "Watcher stopped"
```

**Verification after rollback:**
```bash
# Check watcher has stopped
ps aux | grep omni-watcher

# Verify restored file
cat ./docs/restored-file.md | head -5

# Restart watcher in preview mode to ensure no pending changes
omni-watcher --watch ./docs --preview-only --log-level info
```