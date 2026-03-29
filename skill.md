---
name: youtube-analyzer
description: "Analyze a YouTube video and provide a structured overview. Use when the user shares a YouTube link, says 'analyze this video', 'summarize this YouTube video', 'break down this course', 'what does this video cover', or asks for notes on a YouTube video. Also handles 'expand on chapter N' for deep-dive into previously analyzed videos."
argument-hint: "[youtube-url]"
---

# YouTube Video Analyzer

Extracts the transcript from any YouTube video and delivers a structured, actionable overview. Handles videos from 1 minute to 10+ hours using a chapter-based pipeline with parallel sub-agent analysis.

**Three transcript layers:** captions first, yt-dlp subtitles second, local Whisper third.
**Three analysis modes:** single-pass (under 30 min), sequential chapters (30 min to 2 hr), parallel sub-agent fan-out (2+ hr).

## Activation

- User shares a YouTube URL (youtube.com/watch, youtu.be, or video ID)
- User asks to analyze, summarize, or break down a YouTube video
- User asks for notes or key takeaways from a video
- User says "expand on chapter N" or "go deeper into [section name]" for a previously analyzed video

---

## Step 0: User Profile (First-Time Onboarding)

Before the first analysis, check if a user profile exists:

```bash
cat ~/.youtube-analyzer-profile.json 2>/dev/null || echo "NO_PROFILE"
```

If `NO_PROFILE`, run the onboarding flow. Ask the user these 5 questions one at a time. Wait for each answer before asking the next:

1. **"What's your name?"**
2. **"What does your business or work involve?"** (industry, services, target customers, role)
3. **"What topics are you most focused on learning about right now?"** (e.g., marketing, AI, coding, sales, design, finance)
4. **"What's the biggest challenge you're trying to solve right now?"**
5. **"Any specific tools, frameworks, or platforms you use daily?"** (e.g., n8n, React, Shopify, HubSpot)

After collecting all 5 answers, save the profile:

```bash
cat > ~/.youtube-analyzer-profile.json << 'PROFILE_EOF'
{
  "name": "USER_NAME",
  "business": "THEIR_ANSWER",
  "learning_focus": "THEIR_ANSWER",
  "current_challenge": "THEIR_ANSWER",
  "tools_and_stack": "THEIR_ANSWER"
}
PROFILE_EOF
```

This profile is used in the "Relevance to You" section of every analysis. The user can update it anytime by saying "update my YouTube analyzer profile."

If a profile already exists, read it silently and continue. Do NOT ask the questions again.

---

## Step 1: Extract Video ID and Check Cache

Parse the YouTube URL to get the video ID. Supported formats:
- `https://www.youtube.com/watch?v=VIDEO_ID`
- `https://youtu.be/VIDEO_ID`
- `https://youtube.com/watch?v=VIDEO_ID&t=123`
- Raw video ID (11 characters)

Then check for existing cached analysis:

```bash
mkdir -p /tmp/yt_VIDEO_ID
if [ -f /tmp/yt_VIDEO_ID/master_analysis.md ]; then
    echo "CACHE_HIT: Master analysis exists"
elif [ -d /tmp/yt_VIDEO_ID/analyses ]; then
    echo "PARTIAL_CACHE: Some chapter analyses exist"
    ls /tmp/yt_VIDEO_ID/analyses/
elif [ -d /tmp/yt_VIDEO_ID/transcripts ]; then
    echo "TRANSCRIPT_CACHE: Transcripts exist, no analyses yet"
    ls /tmp/yt_VIDEO_ID/transcripts/
else
    echo "NO_CACHE: Starting fresh"
fi
```

**If CACHE_HIT:** Read and display `/tmp/yt_VIDEO_ID/master_analysis.md`. Done. Skip all other steps.
**If PARTIAL_CACHE or TRANSCRIPT_CACHE:** Resume from where we left off (skip completed steps).
**If NO_CACHE:** Continue to Step 2.

---

## Step 2: Fetch Video Metadata

Use `yt-dlp --dump-json` to get structured metadata including chapters:

```bash
yt-dlp --skip-download --dump-json "https://www.youtube.com/watch?v=VIDEO_ID" 2>/dev/null > /tmp/yt_VIDEO_ID/metadata.json
```

Then extract the key fields:

```bash
python3 -c "
import json
with open('/tmp/yt_VIDEO_ID/metadata.json') as f:
    data = json.load(f)
print(f\"TITLE: {data.get('title', 'Unknown')}\")
print(f\"CHANNEL: {data.get('channel', 'Unknown')}\")
print(f\"DURATION: {data.get('duration_string', 'Unknown')}\")
print(f\"DURATION_SECS: {data.get('duration', 0)}\")
chapters = data.get('chapters') or []
print(f\"CHAPTERS_COUNT: {len(chapters)}\")
if chapters:
    for ch in chapters:
        print(f\"  CH: {ch['start_time']}-{ch.get('end_time', '?')} | {ch['title']}\")
print('---DESCRIPTION---')
print(data.get('description', '')[:3000])
"
```

Record the duration in seconds. This determines the analysis pipeline.

---

## Step 3: Chapter Detection (Three-Tier Fallback)

Chapters are the atomic unit of work. Every operation (transcribe, analyze, cache, expand) works per-chapter.

Run this Python script to detect and write chapters:

```bash
python3 << 'PYEOF'
import json, re, os

VIDEO_ID = "VIDEO_ID"
base = f"/tmp/yt_{VIDEO_ID}"

with open(f"{base}/metadata.json") as f:
    data = json.load(f)

duration = data.get("duration", 0)
chapters = []

# Tier 1: YouTube native chapters
yt_chapters = data.get("chapters") or []
if yt_chapters:
    for i, ch in enumerate(yt_chapters):
        end = yt_chapters[i+1]["start_time"] if i+1 < len(yt_chapters) else duration
        chapters.append({
            "index": i+1,
            "title": ch["title"].strip(),
            "start": int(ch["start_time"]),
            "end": int(end)
        })
    print(f"Tier 1: Found {len(chapters)} YouTube chapters")

# Tier 2: Description timestamp parsing
if not chapters:
    desc = data.get("description", "")
    pattern = r'(?:(\d{1,2}):)?(\d{1,2}):(\d{2})\s+(.+)'
    matches = re.findall(pattern, desc)
    if len(matches) >= 3:
        for i, m in enumerate(matches):
            hours = int(m[0]) if m[0] else 0
            mins = int(m[1])
            secs = int(m[2])
            start = hours * 3600 + mins * 60 + secs
            end = duration
            if i+1 < len(matches):
                nh = int(matches[i+1][0]) if matches[i+1][0] else 0
                end = nh * 3600 + int(matches[i+1][1]) * 60 + int(matches[i+1][2])
            chapters.append({
                "index": i+1,
                "title": m[3].strip(),
                "start": start,
                "end": end
            })
        print(f"Tier 2: Parsed {len(chapters)} chapters from description")

# Tier 3: Synthetic 15-minute segments
if not chapters:
    segment_length = 900  # 15 minutes
    num_segments = max(1, -(-duration // segment_length))  # ceiling division
    for i in range(num_segments):
        start = i * segment_length
        end = min((i+1) * segment_length, duration)
        sm, ss = divmod(start, 60)
        sh, sm = divmod(sm, 60)
        em, es = divmod(end, 60)
        eh, em = divmod(em, 60)
        chapters.append({
            "index": i+1,
            "title": f"Segment {i+1} ({sh:01d}:{sm:02d}:{ss:02d} - {eh:01d}:{em:02d}:{es:02d})",
            "start": start,
            "end": end
        })
    print(f"Tier 3: Created {len(chapters)} synthetic segments")

# Validate: split chapters longer than 20 minutes (for Whisper timeout safety)
MAX_CHAPTER_SECS = 1200  # 20 minutes
validated = []
idx = 1
for ch in chapters:
    length = ch["end"] - ch["start"]
    if length <= MAX_CHAPTER_SECS:
        ch["index"] = idx
        validated.append(ch)
        idx += 1
    else:
        num_parts = -(-length // MAX_CHAPTER_SECS)
        part_len = length // num_parts
        for p in range(num_parts):
            sub_start = ch["start"] + p * part_len
            sub_end = ch["start"] + (p+1) * part_len if p+1 < num_parts else ch["end"]
            suffix = f" (Part {p+1})" if num_parts > 1 else ""
            validated.append({
                "index": idx,
                "title": f"{ch['title']}{suffix}",
                "start": sub_start,
                "end": sub_end
            })
            idx += 1

# Merge chapters shorter than 60 seconds with next chapter
merged = []
i = 0
while i < len(validated):
    ch = validated[i].copy()
    while (ch["end"] - ch["start"]) < 60 and i+1 < len(validated):
        i += 1
        ch["end"] = validated[i]["end"]
    merged.append(ch)
    i += 1

# Re-index and add slugs
for i, ch in enumerate(merged):
    ch["index"] = i + 1
    slug = re.sub(r'[^a-z0-9]+', '-', ch["title"].lower()).strip('-')[:40]
    ch["slug"] = f"ch{ch['index']:02d}_{slug}"

with open(f"{base}/chapters.json", "w") as f:
    json.dump(merged, f, indent=2)

print(f"\nFinal: {len(merged)} chapters written to chapters.json")
for ch in merged:
    m1, s1 = divmod(ch["start"], 60)
    h1, m1 = divmod(m1, 60)
    m2, s2 = divmod(ch["end"], 60)
    h2, m2 = divmod(m2, 60)
    print(f"  [{ch['index']:2d}] {h1:01d}:{m1:02d}:{s1:02d} - {h2:01d}:{m2:02d}:{s2:02d}  {ch['title']}")
PYEOF
```

---

## Step 4: Route by Duration

Read the duration from metadata and route:

| Duration | Pipeline | Reason |
|----------|----------|--------|
| Under 30 min | **Single-pass.** Fetch full transcript, analyze in one shot. | Transcript fits in context. No chapters needed. |
| 30 min to 2 hours | **Sequential chapters.** Per-chapter transcript + analysis, done in-context sequentially. | Manageable number of chapters (3-8). Sub-agents overhead not worth it. |
| Over 2 hours | **Parallel sub-agent fan-out.** Per-chapter transcript, then fan out analysis to parallel sub-agents (Sonnet), then synthesize in main context. | Too many chapters for sequential. Fan-out saves time. |

For single-pass (under 30 min), use the legacy pipeline in Step 5-Legacy below.
For chapter-based pipelines (30+ min), continue to Step 5.

---

## Step 5-Legacy: Single-Pass Pipeline (Under 30 Minutes)

This is the original pipeline for short videos. No chapters needed.

### Fetch transcript

**Layer 1: Captions (instant)**

```bash
python3 -c "
import warnings
warnings.filterwarnings('ignore')
from youtube_transcript_api import YouTubeTranscriptApi
ytt_api = YouTubeTranscriptApi()
transcript = ytt_api.fetch('VIDEO_ID')
for entry in transcript:
    minutes = int(entry.start // 60)
    seconds = int(entry.start % 60)
    print(f'[{minutes:02d}:{seconds:02d}] {entry.text}')
" > /tmp/yt_VIDEO_ID/full_transcript.txt 2>/dev/null
```

If Layer 1 fails, try listing available languages and fetch with the available language code.

**Layer 2: yt-dlp subtitles**

```bash
yt-dlp --skip-download --write-auto-sub --write-sub --sub-lang "en" --sub-format vtt --convert-subs srt -o "/tmp/yt_VIDEO_ID/subs" "https://www.youtube.com/watch?v=VIDEO_ID" 2>/dev/null
```

Then parse the SRT file and write to `/tmp/yt_VIDEO_ID/full_transcript.txt`.

**Layer 3: Whisper (local, free)**

```bash
yt-dlp -x --audio-format mp3 --audio-quality 5 -o "/tmp/yt_VIDEO_ID/full_audio.%(ext)s" "https://www.youtube.com/watch?v=VIDEO_ID"
```

```bash
/opt/homebrew/bin/python3.13 -c "
import mlx_whisper
result = mlx_whisper.transcribe('/tmp/yt_VIDEO_ID/full_audio.mp3', path_or_hf_repo='mlx-community/whisper-large-v3-turbo', word_timestamps=True)
for seg in result['segments']:
    minutes = int(seg['start'] // 60)
    seconds = int(seg['start'] % 60)
    print(f'[{minutes:02d}:{seconds:02d}] {seg[\"text\"].strip()}')
" > /tmp/yt_VIDEO_ID/full_transcript.txt
```

Clean up audio: `rm -f /tmp/yt_VIDEO_ID/full_audio.mp3`

### Analyze and output

Read `/tmp/yt_VIDEO_ID/full_transcript.txt` and produce the standard output format (see Output Format section below). Save to `/tmp/yt_VIDEO_ID/master_analysis.md`. Done.

---

## Step 5: Chapter-Based Transcript Acquisition (30+ Minutes)

### Layer 1: Captions (instant, then split by chapter)

Fetch full captions and split into per-chapter files in one script:

```bash
python3 << 'PYEOF'
import warnings, json
warnings.filterwarnings('ignore')
from youtube_transcript_api import YouTubeTranscriptApi

VIDEO_ID = "VIDEO_ID"
base = f"/tmp/yt_{VIDEO_ID}"

import os
os.makedirs(f"{base}/transcripts", exist_ok=True)

with open(f"{base}/chapters.json") as f:
    chapters = json.load(f)

ytt_api = YouTubeTranscriptApi()
transcript = ytt_api.fetch(VIDEO_ID)
entries = [{"start": e.start, "text": e.text} for e in transcript]

for ch in chapters:
    ch_entries = [e for e in entries if ch["start"] <= e["start"] < ch["end"]]
    filepath = f"{base}/transcripts/{ch['slug']}.txt"
    with open(filepath, "w") as f:
        for e in ch_entries:
            total = int(e["start"])
            mins, secs = divmod(total, 60)
            hrs, mins = divmod(mins, 60)
            f.write(f"[{hrs:01d}:{mins:02d}:{secs:02d}] {e['text']}\n")
    print(f"  Wrote {len(ch_entries)} lines to {ch['slug']}.txt")

print(f"\nDone: {len(chapters)} chapter transcripts saved")
PYEOF
```

If Layer 1 succeeds, skip to Step 6.

### Layer 2: yt-dlp Subtitles (free, fast fallback)

If captions API fails, try yt-dlp subtitle extraction:

```bash
yt-dlp --skip-download --write-auto-sub --write-sub --sub-lang "en" --sub-format vtt --convert-subs srt -o "/tmp/yt_VIDEO_ID/subs" "https://www.youtube.com/watch?v=VIDEO_ID" 2>/dev/null
```

If an SRT file is produced, parse it and split by chapter timestamps into per-chapter transcript files (same structure as Layer 1 output). Use a Python script to parse the SRT, extract timestamps and text, then split based on chapters.json.

If Layer 2 succeeds, skip to Step 6.

### Layer 3: Whisper (local, handles anything)

Only reach this if both captions and yt-dlp subtitles fail.

**IMPORTANT for long videos:** Before starting Whisper, calculate estimated time and inform the user:

```
This [DURATION] video has [N] chapters. Whisper transcription will take approximately [ESTIMATE].
Each chapter takes ~3-5 minutes to transcribe locally.

Options:
1. Transcribe all chapters (full analysis)
2. Select specific chapters to transcribe
3. Cancel
```

For option 2, the user provides chapter numbers and only those are processed.

**Step 5.3a: Download full audio**

```bash
yt-dlp -x --audio-format mp3 --audio-quality 5 -o "/tmp/yt_VIDEO_ID/full_audio.%(ext)s" "https://www.youtube.com/watch?v=VIDEO_ID"
```

**Step 5.3b: Slice audio per chapter with ffmpeg**

```bash
python3 << 'PYEOF'
import json, subprocess, os

VIDEO_ID = "VIDEO_ID"
base = f"/tmp/yt_{VIDEO_ID}"
os.makedirs(f"{base}/audio_chunks", exist_ok=True)

with open(f"{base}/chapters.json") as f:
    chapters = json.load(f)

for ch in chapters:
    outpath = f"{base}/audio_chunks/{ch['slug']}.mp3"
    if os.path.exists(outpath):
        print(f"  SKIP (exists): {ch['slug']}.mp3")
        continue
    cmd = [
        "ffmpeg", "-y", "-i", f"{base}/full_audio.mp3",
        "-ss", str(ch["start"]), "-to", str(ch["end"]),
        "-c", "copy", outpath
    ]
    subprocess.run(cmd, capture_output=True)
    dur = ch["end"] - ch["start"]
    print(f"  Sliced: {ch['slug']}.mp3 ({dur}s)")

print(f"\nDone: {len(chapters)} audio chunks created")
PYEOF
```

**Step 5.3c: Transcribe each chapter sequentially**

CRITICAL: Run Whisper one chapter at a time. Each command is a separate Bash call. Do NOT run multiple Whisper instances in parallel (GPU memory contention will crash).

For each chapter, check cache first, then transcribe:

```bash
SLUG="ch01_chapter-name"
VIDEO_ID="VIDEO_ID"

if [ -f "/tmp/yt_${VIDEO_ID}/transcripts/${SLUG}.txt" ]; then
    echo "CACHED: ${SLUG}"
else
    mkdir -p "/tmp/yt_${VIDEO_ID}/transcripts"
    /opt/homebrew/bin/python3.13 -c "
import mlx_whisper
result = mlx_whisper.transcribe('/tmp/yt_${VIDEO_ID}/audio_chunks/${SLUG}.mp3', path_or_hf_repo='mlx-community/whisper-large-v3-turbo', word_timestamps=True)
offset = CHAPTER_START_SECONDS
with open('/tmp/yt_${VIDEO_ID}/transcripts/${SLUG}.txt', 'w') as f:
    for seg in result['segments']:
        total = int(seg['start'] + offset)
        hrs = total // 3600
        mins = (total % 3600) // 60
        secs = total % 60
        f.write(f'[{hrs}:{mins:02d}:{secs:02d}] {seg[\"text\"].strip()}\n')
print('Done: ${SLUG}')
"
fi
```

Replace `SLUG`, `VIDEO_ID`, and `CHAPTER_START_SECONDS` for each chapter.

After each chapter completes, report progress to the user:
"Transcribed chapter [N]/[TOTAL]: [Chapter Title] ([duration])"

**Step 5.3d: Clean up audio files**

After all chapters are transcribed:

```bash
rm -rf /tmp/yt_VIDEO_ID/audio_chunks /tmp/yt_VIDEO_ID/full_audio.mp3
```

---

## Step 6: Chapter Analysis

Read `chapters.json` to determine how many chapters exist.

### Route A: Sequential Analysis (under 8 chapters)

For each chapter, read the transcript file and produce a per-chapter analysis. Write each to `/tmp/yt_VIDEO_ID/analyses/chNN_analysis.md`.

Process sequentially in the main context:

1. Read `/tmp/yt_VIDEO_ID/transcripts/ch01_slug.txt`
2. Analyze using the Chapter Analysis Template (below)
3. Write result to `/tmp/yt_VIDEO_ID/analyses/ch01_analysis.md`
4. Repeat for next chapter

### Route B: Parallel Sub-Agent Fan-Out (8+ chapters)

Dispatch analysis to parallel sub-agents using the Agent tool. Each sub-agent gets one chapter.

**Dispatch in batches of 4-5 sub-agents at a time.** Wait for batch to complete before dispatching next batch.

Each sub-agent prompt:

```
You are analyzing one chapter of a YouTube video for a structured breakdown.

Video: [TITLE] by [CHANNEL]
Chapter [N]/[TOTAL]: [CHAPTER TITLE]
Time range: [START] - [END]

Read the transcript at: /tmp/yt_VIDEO_ID/transcripts/SLUG.txt

Analyze it and write your analysis to: /tmp/yt_VIDEO_ID/analyses/chNN_analysis.md

Use this exact format for your analysis:

## Chapter [N]: [Chapter Title]
**Time range:** [START] - [END]
**Duration:** [X]m [Y]s

### Summary
[2-3 sentences on what this chapter covers and its core point]

### Key Points
[Bullet points of the most important ideas, frameworks, or claims]

### Notable Quotes
[1-2 direct quotes with timestamps that capture the most impactful moments]
> "Quote" - (timestamp)

### Frameworks/Tools Mentioned
[Any named frameworks, tools, products, or methodologies referenced]

### Action Items
[Concrete takeaways that can be applied immediately]

Rules:
- Preserve the speaker's actual terminology. Don't genericize.
- Be direct. No filler.
- Timestamps must match the actual video positions.
```

Use `model: "sonnet"` for sub-agents (mechanical analysis task). Reserve the main context for synthesis.

After all sub-agents complete, verify all analysis files exist:

```bash
ls /tmp/yt_VIDEO_ID/analyses/ | wc -l
```

If any are missing, re-dispatch those specific chapters.

---

## Step 7: Master Synthesis

Read all per-chapter analysis files and synthesize into the master overview.

```bash
for f in /tmp/yt_VIDEO_ID/analyses/ch*_analysis.md; do echo "=== $(basename $f) ==="; cat "$f"; echo; done
```

Read this combined output (each chapter analysis is ~200-400 words, so even 40 chapters totals ~8,000-16,000 words, fits in context).

Also read the user profile for personalization:

```bash
cat ~/.youtube-analyzer-profile.json 2>/dev/null
```

Produce the master analysis using the Master Output Format below. Write it to `/tmp/yt_VIDEO_ID/master_analysis.md`.

Display the master analysis to the user.

---

## Output Format: Master Analysis

```
# [Video Title]
**Channel:** [channel name]
**Duration:** [duration]
**Video:** [original URL]
**Transcript source:** [Captions | yt-dlp Subtitles | Whisper transcription]
**Chapters analyzed:** [N]

---

## TL;DR
[2-3 sentence summary of the entire video arc and core takeaway]

## Chapter Map
| # | Chapter | Time | Core Point |
|---|---------|------|------------|
| 1 | [Title] | 0:00:00 - 0:05:30 | [One-line summary] |
| 2 | [Title] | 0:05:30 - 0:12:00 | [One-line summary] |

## Core Insights & Takeaways
[Synthesized across ALL chapters. Deduplicated. Ordered by importance, not chronology.]

- **[Insight]:** Explanation
- **[Insight]:** Explanation

## Frameworks & Tools Mentioned
[Aggregated from all chapters. Deduplicated.]

- **[Framework/Tool]:** What it is and how it was used/referenced

## Action Items
[Aggregated from all chapters. Deduplicated. Ordered by impact.]

1. Step one
2. Step two

## Notable Quotes
[Best 4-6 across all chapters]

> "Quote here" - (timestamp)

## Narrative Arc
[How the video's argument/teaching builds across chapters. What is the progression? Where does it shift?]

## Relevance to You
[PERSONALIZED SECTION. Read the user profile from ~/.youtube-analyzer-profile.json. Connect specific insights from the video to the user's business, current challenges, learning focus, and tools. Be specific. Reference their actual situation.

If no profile exists, include a generic "How to Apply This" section instead with universally applicable takeaways.]

---

*Full transcripts saved to `/tmp/yt_VIDEO_ID/transcripts/`. Say "expand on chapter N" or "expand on [topic]" for a deep-dive into any section.*
```

For single-pass videos (under 30 min), use the simpler format without Chapter Map or Narrative Arc. Include Key Topics with timestamps instead.

---

## Step 8: Expand Capability

When the user asks to expand on a section after an initial analysis:

**Trigger phrases:**
- "expand on chapter N"
- "go deeper into [chapter title]"
- "what exactly did they say about [topic]?"
- "more detail on the [section name] part"

### Expand by chapter number or title

1. Read `/tmp/yt_VIDEO_ID/chapters.json`
2. Match the user's request to a chapter by number or fuzzy title match
3. Read the full transcript: `/tmp/yt_VIDEO_ID/transcripts/SLUG.txt`
4. Also read `~/.youtube-analyzer-profile.json` for personalized context
5. Produce a deep-dive analysis:

```
## Deep Dive: Chapter [N] - [Title]
**Time range:** [START] - [END]
**Duration:** [X]m [Y]s

### Detailed Breakdown

[Minute-by-minute walkthrough of what is covered. Every distinct argument, claim, example, and transition.]

#### [Sub-topic at timestamp]
[What was said, what was demonstrated, what was argued]

#### [Sub-topic at timestamp]
[What was said, what was demonstrated, what was argued]

### Every Key Claim Made
[Numbered list of every substantive claim or teaching point, with timestamps]

1. [Claim] (timestamp)
2. [Claim] (timestamp)

### All Examples & Evidence Cited
[Every example, case study, statistic, or piece of evidence mentioned]

### Direct Quotes
[All notable quotes from this chapter, with timestamps]

> "Quote" - (timestamp)

### Tools, Resources & Links Mentioned
[Everything referenced: tools, websites, books, people, products]

### How This Applies to You
[Read the user profile. Connect this chapter's specific content to their business, challenges, and goals. Be concrete. If the chapter teaches a technique, explain how they could implement it given their stack and situation.

If no profile exists, provide general application guidance instead.]
```

### Expand by topic (cross-chapter)

If the user asks about a topic rather than a specific chapter:

1. Search all chapter transcripts for relevant terms:
```bash
grep -li "SEARCH_TERM" /tmp/yt_VIDEO_ID/transcripts/*.txt
```

2. Read the matching sections from each chapter transcript
3. Synthesize a cross-chapter analysis of that specific topic

```
## Topic Deep Dive: [Topic]

### Mentions Across Chapters
[Which chapters discuss this topic, with timestamps]

### What Was Said
[Synthesized view of everything said about this topic across the full video]

### Key Quotes on This Topic
> "Quote" - (Chapter N, timestamp)

### How This Applies to You
[Personalized application based on user profile]
```

---

## Profile Management

When the user says "update my profile" or "update my YouTube analyzer profile":

1. Read the current profile: `cat ~/.youtube-analyzer-profile.json`
2. Show them their current profile
3. Ask which fields they want to update
4. Save the updated profile

When the user says "reset my profile" or "delete my YouTube analyzer profile":

```bash
rm ~/.youtube-analyzer-profile.json
```

The next analysis will trigger the onboarding questions again.

---

## Rules

1. Always fetch the transcript first. Never summarize from title/description alone.
2. Try Layer 1 (captions) first, then Layer 2 (yt-dlp subtitles), then Layer 3 (Whisper). Only escalate when the previous layer fails.
3. Preserve the speaker's actual terminology and frameworks. Don't genericize.
4. Timestamps must map to real positions in the video so the user can jump to specific sections.
5. If the video is a course or tutorial, break it into clear modules/lessons.
6. If the video is a conversation/podcast, identify each speaker's key contributions.
7. Keep the analysis direct and results-focused. No filler.
8. For Whisper transcriptions, note that timestamps may have slight drift (~1-2 seconds).
9. For long videos (2+ hr) using Whisper, always warn the user about estimated transcription time and offer selective chapter transcription.
10. Always save transcripts and analyses to disk. Never discard them after analysis.
11. When dispatching sub-agents for parallel analysis, use `model: "sonnet"` for chapter analysis and reserve the main context for synthesis.
12. Run Whisper transcription sequentially (one chapter at a time). Never parallel. GPU memory contention will crash.
13. After completing analysis, always remind the user they can expand on specific chapters.
14. For the expand capability, read from cached transcript files on disk. Never re-fetch or re-transcribe.
15. Always read the user profile for the personalized "Relevance to You" section. If no profile exists, use generic application guidance and suggest the user set one up.

## Handling Different Video Types

| Video Type | Emphasis |
|-----------|----------|
| Course/Tutorial | Module breakdown, step-by-step process, tools mentioned, learning progression |
| Podcast/Interview | Speaker contributions, key debates, frameworks shared, points of agreement/disagreement |
| Conference Talk | Core thesis, supporting evidence, call to action, Q&A insights |
| Product Demo | Features shown, use cases, pricing/access info, limitations mentioned |
| Vlog/Commentary | Main arguments, evidence cited, conclusions, counterarguments addressed |

## Dependencies

| Tool | Purpose | Install |
|------|---------|---------|
| yt-dlp | Video metadata, subtitles, audio download | `brew install yt-dlp` |
| youtube-transcript-api | Caption extraction (Layer 1) | `pip3 install youtube-transcript-api` |
| ffmpeg | Audio chunking for Whisper pipeline | `brew install ffmpeg` |
| mlx-whisper | Local transcription on Apple Silicon (Layer 3) | `/opt/homebrew/bin/python3.13 -m pip install mlx-whisper --break-system-packages` |
| python3 | Chapter detection, transcript splitting | Pre-installed on macOS |
