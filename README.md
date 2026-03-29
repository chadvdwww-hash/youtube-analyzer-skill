# YouTube Analyzer Skill for Claude Code

Analyze any YouTube video from 1 minute to 10+ hours. Get structured breakdowns, key insights, action items, and personalized takeaways. Then expand on any section for a deep dive.

## What It Does

Drop a YouTube link into Claude Code and get a full analysis: chapter map, core insights, action items, notable quotes, and a personalized "Relevance to You" section based on your business and goals.

**Works on any video length.** Short videos get instant single-pass analysis. Long courses and podcasts get chapter-by-chapter breakdowns with parallel processing.

### How It Handles Long Videos

| Duration | Strategy |
|----------|----------|
| Under 30 min | Single-pass analysis. Instant. |
| 30 min to 2 hours | Sequential chapter-by-chapter analysis. |
| Over 2 hours | Parallel sub-agent fan-out. Multiple chapters analyzed simultaneously. |

### Three Transcript Layers

1. **Captions API** (instant, free, works ~95% of the time)
2. **yt-dlp subtitles** (fallback for when captions API fails)
3. **Local Whisper** (runs on your machine via mlx-whisper. No API costs. Handles videos with subtitles disabled.)

### Personalized Analysis

On first use, the skill asks 5 quick questions about you and your work. Every analysis after that includes a "Relevance to You" section connecting the video's insights to your specific situation.

### Expand on Demand

After any analysis, say:
- "Expand on chapter 5" for a minute-by-minute deep dive
- "What did they say about pricing?" for cross-chapter topic search
- "Go deeper into the marketing section" for fuzzy title matching

All transcripts are cached to disk, so expansions are instant (no re-downloading or re-transcribing).

## Install

### Prerequisites

```bash
brew install yt-dlp ffmpeg
pip3 install youtube-transcript-api
```

For the Whisper fallback (Apple Silicon only):

```bash
/opt/homebrew/bin/python3.13 -m pip install mlx-whisper --break-system-packages
```

### Install the Skill

```bash
npx skills add chadvanderwal/youtube-analyzer-skill
```

Or manually: copy `skill.md` to `~/.claude/skills/youtube-analyzer/skill.md`

## Usage

Just share a YouTube URL in Claude Code:

```
Analyze this: https://www.youtube.com/watch?v=VIDEO_ID
```

Or use the slash command:

```
/youtube-analyzer https://www.youtube.com/watch?v=VIDEO_ID
```

### After Analysis

```
Expand on chapter 3
```

```
What did they say about AI agents across the whole video?
```

```
Go deeper into the deployment section
```

### Profile Management

```
Update my YouTube analyzer profile
```

```
Reset my profile
```

## How It Works

1. Fetches video metadata (title, channel, duration, chapters)
2. Detects chapters (YouTube chapters > description timestamps > auto-generated 15min segments)
3. Acquires transcript per-chapter (captions > yt-dlp subs > Whisper)
4. Analyzes each chapter (sequentially or via parallel sub-agents depending on video length)
5. Synthesizes a master analysis with personalized relevance
6. Caches everything to `/tmp/yt_VIDEO_ID/` for instant expand-on-demand

## Caching

Everything is cached at `/tmp/yt_VIDEO_ID/`:

```
/tmp/yt_VIDEO_ID/
  metadata.json
  chapters.json
  transcripts/
    ch01_introduction.txt
    ch02_core-concepts.txt
  analyses/
    ch01_analysis.md
    ch02_analysis.md
  master_analysis.md
```

If analysis gets interrupted, it resumes from where it left off. Already-transcribed chapters are never re-processed.

Your user profile is saved at `~/.youtube-analyzer-profile.json`.

## License

MIT
