---
name: summarize
description: Summarize any content (YouTube video, article, whitepaper/PDF, podcast episode, book chapter, etc.) into a rich Obsidian note with section-by-section breakdowns, wikilinks to all technical concepts and people, and reference notes for every linked term. Use when the user provides a URL, file, or content to summarize and document in the vault.
user_invocable: true
---

# Summarize

Universal content summarizer. Takes any input — YouTube video, web article, whitepaper/PDF, epub book, podcast, lecture — and produces a rich, interlinked Obsidian summary note with reference notes for every concept mentioned.

## Requirements

**Vault structure** — the skill expects these folders inside your Obsidian vault. Folder names are the defaults; override them in the Configuration block below if your vault uses different names.

| Folder | Purpose |
|---|---|
| `08 Summaries/` | Where summary notes land |
| `07 References/` | Concept / company / product / place notes |
| `04 People/` | Person notes (creators, guests, mentioned people) |
| `02 Daily/YYYY/MM/` | Daily notes, named `MM-DD-YY ddd.md` (e.g. `03-29-26 Sun.md`) |
| `_Templates/` | Note templates — skill installs `new person template.md` here on first run |
| `_Bases/` (optional) | Obsidian Bases — only needed if you use the Bases plugin |

**CLI tools** — install these before first use, or let Step 0 walk you through it:

| Tool | Purpose | Install |
|---|---|---|
| `yt-dlp` | YouTube/podcast download + metadata + subs | `brew install yt-dlp` or `pip install yt-dlp` |
| `defuddle` | Web article extraction | `npm install -g defuddle` |
| `pdftotext` | PDF text extraction | `brew install poppler` |
| `pandoc` | EPUB / DOCX → markdown | `brew install pandoc` |
| `mlx_whisper` (optional) | Local audio transcription fallback | `pip install mlx-whisper` |

Alternative to `mlx_whisper`: set `ELEVENLABS_API_KEY` to use ElevenLabs Scribe for transcription.

## Configuration

The skill reads these variables at runtime. Override any of them via environment variables, or edit the defaults here:

```
AI_VAULT_ROOT     = $AI_VAULT_ROOT        # auto-detected if not set (see Step 0a)
PKM_VAULT_ROOT       = $PKM_VAULT_ROOT          # personal vault root for daily notes; defaults to AI_VAULT_ROOT if not set
SUMMARIES_DIR  = summaries
REFERENCES_DIR = concepts
PEOPLE_DIR     = profiles
DAILY_DIR      = daily
TEMPLATES_DIR  = templates
BASES_DIR      = _Bases
```

All paths below are relative to `$AI_VAULT_ROOT`.

## Trigger

When the user provides content to summarize: a URL (YouTube, article, blog), a PDF/file path, pasted text, or a reference to content already in the vault.

## Inputs

- **Source**: URL, file path, or pasted text
- **Audience** (optional): defaults to "general reader." User may specify (e.g. "high school student", "expert", "5-year-old")
- **Depth** (optional): defaults to "full." User may request "tldr only", "section-by-section", or "deep dive"

## Step 0: Bootstrap check (first run)

Before doing any work, verify the environment is ready. **Skip any check that already passes** — only prompt the user when something is actually missing. Do not re-run Step 0 on subsequent invocations if the initial setup succeeded; you can tell it already ran if `$AI_VAULT_ROOT` resolves and the required folders + tools are present.

### 0a. Resolve the vault root

```bash
vault=""
if [ -n "$AI_VAULT_ROOT" ]; then
  vault="$AI_VAULT_ROOT"
else
  dir="$PWD"
  while [ "$dir" != "/" ]; do
    if [ -d "$dir/.obsidian" ]; then vault="$dir"; break; fi
    dir="$(dirname "$dir")"
  done
fi
echo "Vault: ${vault:-NOT FOUND}"

# PKM_VAULT_ROOT defaults to AI_VAULT_ROOT if not set separately
pkm_root="${PKM_VAULT_ROOT:-$vault}"
echo "PKM (daily notes): ${pkm_root:-NOT FOUND}"
```

If no vault is found, ask the user: **"What's the absolute path to your Obsidian vault?"** Use their answer as `$AI_VAULT_ROOT` for the session (and suggest they set it permanently in their shell profile). If `$PKM_VAULT_ROOT` is not set, it defaults to `$AI_VAULT_ROOT`.

### 0b. Check required folders

```bash
for d in "$SUMMARIES_DIR" "$REFERENCES_DIR" "$PEOPLE_DIR" "$DAILY_DIR" "$TEMPLATES_DIR"; do
  [ -d "$AI_VAULT_ROOT/$d" ] || echo "MISSING: $d"
done
```

For each missing folder, ask the user: **"Create `<folder>` in your vault? [y/N]"** — if yes, `mkdir -p "$AI_VAULT_ROOT/<folder>"`.

### 0c. Check required CLI tools

```bash
for tool in yt-dlp defuddle pdftotext pandoc; do
  command -v "$tool" >/dev/null 2>&1 || echo "MISSING: $tool"
done
```

For each missing tool, tell the user what's missing and **ask before installing** — installs touch the user's system. Use the install commands from the Requirements table above. If the user declines, note which tools are missing and warn that the corresponding content types (YouTube, web articles, PDFs, EPUBs) will fail until installed.

### 0d. Install the person template if missing

The skill ships two person templates in its own `templates/` folder:

- **`new person template.md`** — full version with Dataview callouts (current age, total hours talked) and Obsidian Bases embeds (`posts.base`, `books.base`, `meetings.base`). Requires the Dataview plugin and Obsidian Bases.
- **`new person template (minimal).md`** — stripped version. Just frontmatter, a `> [!info]` summary callout, and an `## updates` section. Works in any vault.

If `$AI_VAULT_ROOT/$TEMPLATES_DIR/new person template.md` already exists, leave it alone — the user may have their own customized version.

Otherwise, ask the user which version to install:

> **Install person template — which version?**
> 1. **Minimal** (default, works in any vault)
> 2. **Full** (requires Dataview plugin + Obsidian Bases)

Then copy the chosen template into the user's `_Templates/` folder:

```bash
skill_dir="$(dirname "$0")"   # or wherever this SKILL.md lives
target="$AI_VAULT_ROOT/$TEMPLATES_DIR/new person template.md"

if [ ! -f "$target" ]; then
  # Use the user's choice — default to minimal
  src="$skill_dir/templates/new person template (minimal).md"
  # if user picked full: src="$skill_dir/templates/new person template.md"
  cp "$src" "$target"
fi
```

Note: whichever version gets installed lands at `_Templates/new person template.md` (no `(minimal)` suffix) so the skill's later references work uniformly.

Once Step 0 passes, proceed to Step 1.

## Step 1: Detect content type and extract text

### YouTube video
```bash
# Get metadata
yt-dlp --cookies-from-browser chrome \
  --print "%(id)s|%(title)s|%(duration)s|%(upload_date)s|%(view_count)s|%(channel)s|%(channel_id)s" \
  --no-download "<URL>"

# Try auto-subtitles first (fastest, free)
yt-dlp --cookies-from-browser chrome \
  --write-auto-sub --sub-lang en --sub-format json3 \
  --skip-download -o "/tmp/summarize/%(id)s" "<URL>"
```

If auto-subs exist, extract text from the JSON3 file. If not, or if quality is poor:
- Download audio and transcribe (same as `youtube-transcribe` skill — ask user: local mlx_whisper or ElevenLabs Scribe)

### Web article / blog post
```bash
defuddle parse "<URL>" --md -o /tmp/summarize/article.md
```

If defuddle is not installed: `npm install -g defuddle`

Extract title, author, date, domain from defuddle metadata:
```bash
defuddle parse "<URL>" -p title
defuddle parse "<URL>" -p domain
```

### PDF
```bash
pdftotext "<path>" /tmp/summarize/paper.txt
```

If `pdftotext` is not available: `brew install poppler`

### EPUB (books)
```bash
# Extract full text as markdown (preserves chapter structure)
pandoc "<path>" -t markdown --wrap=none -o /tmp/summarize/book.md

# If you need chapter boundaries, extract the TOC:
pandoc "<path>" -t json | python3 -c "
import json, sys
doc = json.load(sys.stdin)
for block in doc['blocks']:
    if block['t'] == 'Header':
        level = block['c'][0]
        text = ''.join(
            item['c'] if item['t'] == 'Str' else ' ' if item['t'] == 'Space' else ''
            for item in block['c'][2]
        )
        print(f'L{level}: {text}')
"
```

**Chapter splitting strategy for books:**
1. Extract full text with `pandoc` → markdown
2. Identify chapter boundaries from headers (epubs have built-in TOC structure that pandoc preserves as `#`/`##` headers)
3. Split into one chunk per chapter
4. Dispatch parallel Opus subagents — **one per chapter** — same as any other long content
5. A typical book (60-100k words, 15-30 chapters) produces chapters of ~3-5k words each — well within subagent context limits

**For very long books (>30 chapters):** batch chapters into groups of ~5 per subagent to keep the number of parallel agents manageable. Each subagent summarizes its batch and returns section summaries.

**CRITICAL — Book summary depth requirement:**
- Each chapter MUST get its own dedicated `## Chapter N: Title` section with a **substantial** summary (300-600 words per chapter depending on chapter length)
- Do NOT batch multiple chapters into a single brief paragraph — every chapter gets its own detailed treatment
- Include key arguments, data points, examples, and quotes from each chapter
- A 10-chapter book should produce ~3000-6000 words of summary content (excluding frontmatter/tldr)
- A 30-chapter book should produce ~5000-10000 words
- Think of each chapter summary as a standalone mini-essay that captures the chapter's core contribution
- The goal is that someone reading the summary should understand what each chapter argues, not just what the book is "about" at a high level

**Output structure for books:**
- Location: `08 Summaries/<Book Title>.md` (or `08 Summaries/<Author>/<Book Title>.md` if summarizing multiple books by one author)
- Frontmatter tag: `book`
- Extra fields: `creator` (author wikilink), `published` (year), `isbn` (if known), `source` (wikilink to the epub file if it's in the vault, e.g. `"[[Book Title.epub]]"`)
- Each chapter gets its own `## Chapter N: Title` section in the summary
- Add a `## Chapter Navigation` callout at the top if the book has many chapters

### Other files (txt, docx, etc.)

For `.docx`: `pandoc "<path>" -t markdown --wrap=none -o /tmp/summarize/doc.md`

For plain text: read directly.

### Pasted text / vault note
Read directly from user message or vault path.

## Step 1b: Save transcript (audio/video content only)

For any content that has audio — YouTube videos, podcast episodes, lectures/talks with recordings — save the extracted transcript as a permanent vault note.

**When to create a transcript note:**
- YouTube videos (from auto-subs or whisper transcription)
- Podcast episodes (from transcription)
- Lectures/talks with audio/video recordings
- Any content where the source is spoken word

**Do NOT create transcript notes for:** articles, blog posts, PDFs, books, pasted text — these are already text.

**Location:** Same folder as the summary note, with ` Transcript` appended to the filename.

**Format:**
```markdown
---
date: YYYY-MM-DD
duration: <seconds>
recording: "<source URL>"
meeting: "[[<Summary Note Title>]]"
unread: true
---

[Full timestamped transcript text, one line per segment]
```

**Link from summary:** Add `transcript: "[[<Title> Transcript]]"` to the summary note's frontmatter.

This step happens immediately after text extraction (Step 1) and before output structure planning (Step 2). The transcript is the raw source material — always preserve it.

## Step 2: Determine output structure

Based on content type, choose the appropriate format:

| Content type | Location | Frontmatter tags | Extra fields |
|---|---|---|---|
| YouTube video | `08 Summaries/<Channel>/Summaries/<Title>.md` | `youtube` | `recording`, `views`, `creator`, `people`, `guest`, `hosts`, `guests`, `duration`, `uploaded`, `transcript` |
| Article / blog | `08 Summaries/<Title>.md` | `article` | `creator`, `source` (URL), `published` |
| Whitepaper / PDF | `08 Summaries/<Title>.md` | `paper` | `authors`, `affiliations`, `source` (wikilink to PDF if in vault, or URL), `published` |
| EPUB / book | `08 Summaries/<Title>.md` | `book` | `creator` (author wikilink), `published` (year), `isbn`, `source` (wikilink to epub if in vault) |
| Podcast episode | `08 Summaries/<Show>/Summaries/<Title>.md` | `podcast` | `recording`, `people`, `guest`, `hosts`, `guests`, `duration`, `transcript` |
| Lecture / talk | `08 Summaries/<Title>.md` | `lecture` | `creator`, `recording` (if URL), `transcript` |

**All notes** get: `created`, `updated`, `date`, `summary`, `categories: ["[[posts.base]]"]`, `unread: true`

If a channel/show folder is needed, check if it already exists before creating.

## Step 3: Analyze structure, determine depth, and plan sections

Read the full extracted text. Identify the natural sections/chapters/topics.

### 3a. Determine summary depth from source length

Summary length must be **proportional** to the source material. A 10-minute video and a 3-hour documentary should not produce the same size summary. Use the source word count to determine the target summary word count:

| Source word count | Source examples | Target summary words | Sections | TLDR |
|---|---|---|---|---|
| <1,500 | 5-min video, short article | 200–400 | 1–2 | 2 sentences |
| 1,500–5,000 | 10–20 min video, blog post, short paper | 500–1,200 | 3–5 | 3 sentences |
| 5,000–15,000 | 30–60 min video, long article, whitepaper | 1,500–3,000 | 5–8 | 3–4 sentences |
| 15,000–40,000 | 1–3 hr video/podcast, long paper | 3,000–6,000 | 8–15 | 4–5 sentences |
| 40,000–80,000 | Short book, multi-hour series | 5,000–10,000 | 15–25 | 5 sentences |
| 80,000+ | Full book (200+ pages) | 8,000–15,000 | 20–40 | 5 sentences |

**The ratio is roughly 1:5 to 1:10** — a 10,000-word source should produce ~1,500–2,500 words of summary. Denser/more technical content skews toward the higher end; conversational/repetitive content skews lower.

**For videos/podcasts**, estimate source words from duration: ~150 words/minute for conversational, ~120 words/minute for interviews with pauses, ~170 words/minute for scripted/narrated content. Or just use the actual transcript word count.

**Per-section depth**: each section's word budget should be proportional to its share of the source material. A section covering 20% of the transcript gets ~20% of the summary word budget. Adjust up for particularly dense/important sections, down for filler/repetitive ones.

### 3b. Plan sections and dispatch

**For long content (>3000 source words):** dispatch parallel **Opus** subagents — one per section — to summarize simultaneously. Each subagent gets:
- The section text
- The audience level
- A **specific word count target** (calculated from 3a above)
- Instructions to use `[[wikilinks]]` for every technical concept, person, place, company, and notable noun

**For short content (<3000 source words):** summarize directly without subagents.

**NEVER use Haiku or Sonnet for summarization.** Always Opus.

## Step 4: Assemble the summary note

### Structure

```markdown
---
[frontmatter per Step 2]
---

[embed if applicable: ![[file.pdf]], ```vid URL```, etc.]

> [!tldr]
> [Overview — sentence count per Step 3a depth table. What is it about, who made it, what are the key takeaways?]

## [Section 1 Title]

[Summary paragraphs with [[wikilinks]] to all concepts, people, places, companies, products]

## [Section 2 Title]

[...]

## People Mentioned
- [[Person Name]] — brief context of who they are and their role in this content
```

### Formatting rules

1. **No `# Title` heading** — filename is the title
2. **Never repeat frontmatter in the body** — if it's in metadata, don't write it again
3. **`> [!tldr]`** for the overview, not `## Summary`
4. **`> [!quote]`** callouts for notable quotes (with speaker wikilink and source location if available)
5. **Wikilink EVERYTHING** — people, places, companies, concepts, technical terms, **book/film/show titles**, even if no note exists yet
6. **Use actual Japanese/Chinese characters** for non-English words, not romanization
7. **Timestamps** on topic headings and quotes when available (YouTube, podcasts)
8. **`people` field**: only people who created/appeared in the content. Mentioned people go in `## People Mentioned`
9. **Write summary body in Japanese** — regardless of the source content language, the summary note body (tldr, sections, People Mentioned) must be written in Japanese.
10. **Alias notation for wikilinks in Japanese summaries** — Reference note filenames are always English (see Step 5b). Use alias notation so wikilinks read naturally in Japanese: `[[English Concept Name|日本語のテキスト]]`. Example: 「このイヤホンは[[Harman Target|ハーマンカーブ]]に近い滑らかな[[Frequency Response|周波数特性]]を持ち…」. Use a plain `[[English Concept Name]]` only when the English term reads naturally in the surrounding Japanese text.

### Audience adaptation

- **High school / college student**: plain language, analogies, explain jargon inline before first wikilink use
- **General reader**: balanced — explain key terms but don't over-simplify
- **Expert**: technical language fine, focus on novel contributions and critiques

## Step 5: Create reference notes (one layer deep)

**This is the most important step. Every wikilink MUST resolve to a note. No dangling links.**

### 5a. Extract and audit all wikilinks

After the summary note is fully assembled, extract every unique wikilink programmatically:

```bash
grep -o '\[\[[^]|]*' "<summary_note_path>" | sed 's/\[\[//' | sort -u
```

Then check which ones are missing:

```bash
for term in <each extracted term>; do
  found=$(find "$AI_VAULT_ROOT" -name "$term.md" \
    -not -path "*/.Trash/*" -not -path "*/Clippings/*" 2>/dev/null | head -1)
  if [ -z "$found" ]; then echo "MISSING: $term"; fi
done
```

**Do NOT skip this step. Do NOT estimate from memory which notes exist.** Always run the audit.

### 5b. Create missing notes

#### Technical concepts, companies, products, places
Create in `07 References/<Term>.md`:

- **Filename**: always English (e.g. `Frequency Response.md`, `Harman Target.md`)
- **Body language**: always English — definitions and explanations must be written in English regardless of the source content language. If the source is Japanese, translate the concept description into English before writing it.

```markdown
---
created: YYYY-MM-DDT00:00
updated: YYYY-MM-DDT00:00
type: reference
unread: true
---

[2-4 sentence plain-language explanation in English. Use [[wikilinks]] to cross-reference related concepts.]
```

#### People
Create in `$PEOPLE_DIR/<Full Name>.md` using the person template at `$AI_VAULT_ROOT/$TEMPLATES_DIR/new person template.md` (installed by Step 0d). Conventions:

- **Public figures**: research and write a rich bio (birthday, career, links, key facts). The `> [!info]` callout should be a substantive snapshot — life story, mission, current focus — not a stub.
- **Private individuals**: minimal note with only what's known from the content. The note will grow naturally over time.
- **`> [!note] current age` callout** (from template): keep it if `birthday` is known or can be estimated. If estimated, append `(estimated)` to the callout text — but `birthday` in frontmatter must stay a pure YAML date (e.g. `2001-01-01`), never text.
- **`> [!abstract] total hours talked` callout** (from template): ONLY keep this if the person has had real 1-on-1 calls/meetings with the vault owner (i.e. they appear in meeting notes). Delete the callout for people discovered through summarizing videos, articles, books, or podcasts — those people will never have meeting entries, so the callout would always show 0h.
- **No `# Title` heading** — Obsidian shows the filename as the title.
- **`unread: true`** in frontmatter on every new or modified note.

#### Dispatch in parallel
For large numbers of missing notes (>10), use parallel **Opus** subagents in batches of ~20-25 notes each. Each subagent creates the notes and returns confirmation.

### 5c. Verify — no dangling links

After all notes are created, re-run the audit from 5a to confirm zero missing notes. If any remain (e.g. a subagent failed or skipped one), create them manually. **The summary is not done until this verification passes.**

## Step 6: Update bases (optional — skip if not using Obsidian Bases)

This step only applies if `$AI_VAULT_ROOT/$BASES_DIR/posts.base` exists. If it doesn't, skip Step 6 entirely.

```bash
[ -f "$AI_VAULT_ROOT/$BASES_DIR/posts.base" ] || echo "No posts.base — skipping Step 6"
```

If it does exist:
- **`posts.base`**: if new people appeared as creators/guests, add named views for them using the YAML block below, then embed them in their person notes (in a `## episodes` or `## videos` section) via `![[posts.base#Person Name]]`.
- If a new channel/show folder was created, add a channel-specific view to `posts.base` the same way.

Named view YAML block to append under the `views:` list:
```yaml
  - type: table
    name: "Person Name"
    filters:
      and:
        - recording != null
        - people.contains(link("Person Name"))
    order:
      - date
      - views
      - file.name
      - summary
    sort:
      - property: date
        direction: DESC
```

## Step 7: Update daily note

Append to `$PKM_VAULT_ROOT/$DAILY_DIR/YYYY/MM/DD.md` (e.g. `daily/2026/04/14.md`). If the file doesn't exist, create it with `unread: true` frontmatter. Add the entry under the `### Inputs` section if it exists; otherwise append at the end.

```markdown
### Inputs

- summarized [[Note Title]] — [1-line description of what it is]
```

## Model usage

| Task | Model |
|------|-------|
| Content extraction | Scripts (defuddle, pdftotext, yt-dlp) |
| Section summarization | **Opus** subagents (parallel) |
| Reference note creation | **Opus** subagents (parallel batches) |
| Person note creation | **Opus** |
| **NEVER** | **Haiku or Sonnet** |

## Key rules

1. **Wikilink everything** — every concept, person, company, place, and **book/film/show title** gets a `[[wikilink]]`
2. **One layer deep** — create reference/person notes for EVERY wikilinked term that doesn't already have a note
3. **No `# Title` headings** — Obsidian shows filename as title
4. **Never repeat frontmatter in body** — frontmatter is metadata, body is content
5. **Set `unread: true`** on every note created or modified
6. **Parallel Opus subagents** for long content — one per section for summaries, batches of ~20 for reference notes
7. **Audience-appropriate language** — match the user's requested level
8. **Always embed/link the source** — PDF embed, vid embed, or source URL in frontmatter
9. **`> [!tldr]`** is mandatory — every summary starts with a concise overview callout
10. **Person note `## updates` links to the content note, NEVER the daily note**
11. **Summary body is always Japanese** — source language does not affect output language
