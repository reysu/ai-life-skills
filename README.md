# ai-life-skills

A collection of skills I use with Claude Code to improve my life in various ways. Designed to pair with an AI-managed Obsidian vault — the skills read from and write to the vault.

## Skills

- [`summarize/`](./summarize) — drop in a YouTube video, article, PDF, EPUB, or podcast and it writes a summary note into your vault with wikilinks to every person and concept mentioned. Asks detailed vs minimal mode on each run — detailed creates reference notes for every wikilink, minimal leaves them dangling and saves ~85% tokens.
- [`summarize-call/`](./summarize-call) — drop in a call recording (video or audio) and it transcribes with speaker labels, summarizes, and writes a call note + transcript + person notes for the participants. Same detailed / minimal mode as `summarize`.

(more coming) (daily brief, daily news)

## Install — easy mode

Open Claude Code in any directory and paste this:

> Install the ai-life-skills pack from https://github.com/reysu/ai-life-skills. Clone the repo to `~/src/ai-life-skills`, ask me where I want the new Obsidian vault to live, create the vault folder with the full folder structure the skills expect, and symlink every skill in the repo into `~/.claude/skills/`.

Claude will:
1. Clone the repo
2. Ask where to put the vault (default: `~/ai-vault`)
3. Create the vault folder with the expected structure (see below)
4. Symlink `summarize` and `summarize-call` into `~/.claude/skills/`
5. Copy the person-note template into your vault's `_Templates/` folder

Restart Claude Code so it picks up the new skills. Then run `/summarize` or `/summarize-call`.

> **Recommended: use a new, dedicated Obsidian vault** for these skills rather than your existing personal vault. The skills create and modify many notes/folders automatically, and keeping it separate avoids polluting notes you've written yourself.

### If you already have an Obsidian vault

You can point the skills at an existing vault if you want — tell Claude the path instead of creating a new one and it'll only create any missing folders. Just note the recommendation above about a dedicated vault.

## Install — individual skill only

If you just want one skill and already have a vault:

```bash
mkdir -p ~/src
git clone https://github.com/reysu/ai-life-skills ~/src/ai-life-skills
ln -s ~/src/ai-life-skills/summarize ~/.claude/skills/summarize
# or:
ln -s ~/src/ai-life-skills/summarize-call ~/.claude/skills/summarize-call
```

The skills share a `templates/` folder at the repo root — leave it where it is, both skills reference it with a relative path.

## Usage

### Summarize a video, article, PDF, or book

```
/summarize https://youtube.com/watch?v=...
```

Summary length is proportional to source length — a 10-minute video gets a short summary, a 3-hour podcast gets a long one, a 600-page book gets chapter-by-chapter treatment. Quotes and a link to the transcript go in the frontmatter.

For a book or PDF, drop the file into your vault first (I use `_Attachments/`) then call the skill with the filename:

```
/summarize The Singularity Is Near.epub
```

### Summarize a call

Drop the recording anywhere and call the skill with the path:

```
/summarize-call ~/Downloads/call-with-alex.mp4
```

It'll ask whether to transcribe locally (free, private, slower — walks you through installing whisper and pyannote if you don't have them) or with ElevenLabs Scribe (paid, faster, one API call). Then it writes a call note, the transcript, and person notes for the participants.

### Depth flags (both skills)

Both skills ask "detailed or minimal?" on each run. To skip the prompt — useful for scheduled tasks, `/loop`, or just typing faster — pass the mode in the invocation:

```
/summarize https://youtube.com/... minimal
/summarize https://youtube.com/... detailed
/summarize-call ~/call.mp4 --minimal
/summarize-call ~/call.mp4 -d
```

Accepted tokens:
- **Minimal**: `minimal`, `fast`, `quick`, `--minimal`, `-m`
- **Detailed**: `detailed`, `deep`, `full`, `--detailed`, `-d`

If neither is present, the skill prompts interactively. There's no default — you either pass it or pick when asked.

**What the modes do**:
- **Detailed**: creates reference notes for every wikilink in the output, person notes for every person mentioned (researches public figures), uses the highest-quality model available
- **Minimal**: writes the summary/call note only, leaves wikilinks dangling, creates person notes for creators/guests (or call participants) only, uses Sonnet — saves ~85% on tokens for typical content

## Vault structure

```
your vault/
├── 01 Updates/
├── 02 Daily/YYYY/MM/   # daily notes, named MM-DD-YY ddd.md
├── 03 Meetings/        # call notes + transcripts
├── 04 People/
├── 05 Projects/
├── 06 Research/
├── 07 References/
├── 08 Summaries/
├── _Templates/
├── _Attachments/       # drop ebooks/PDFs here before telling claude to summarize them
└── _Bases/             # optional, only if you use Obsidian Bases
```

Easy-mode install creates this structure for you. If you're using an existing vault, the skills prompt before creating any missing folders on first run. You can also rename any of them in the Configuration block at the top of each SKILL.md.

If you run Claude Code from outside your vault, set `VAULT_ROOT`:

```bash
export VAULT_ROOT="/path/to/vault"
```

Otherwise the skills walk up from your current directory looking for `.obsidian/`.

## Requirements

- `summarize` uses `yt-dlp`, `defuddle`, `pdftotext`, and `pandoc`
- `summarize-call` uses `ffmpeg` (always), plus either `mlx_whisper` + `pyannote.audio` (local path) or an `ELEVENLABS_API_KEY` (cloud path)

Each skill checks what's missing on first run and asks before installing anything.

## Tested on

- macOS 15 (Darwin 25) on Apple Silicon, Python 3.11+, Claude Code CLI
- Local transcription path assumes a Mac with MPS; the skill auto-detects CUDA / MPS / CPU and falls back to CPU on unsupported devices (slower but functional)
- ElevenLabs Scribe path works on any OS with Python + `requests`

## License

MIT.
