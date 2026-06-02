# Daily YouTube Digest

Start your day with a personalized summary of new videos from your favorite YouTube channels — no more missing content from creators you actually want to follow.

## Pain Point

YouTube notifications are unreliable. You subscribe to channels, but their new videos never show up in your home feed. They're not in notifications. They just... disappear. This doesn't mean you don't want to see them — it means YouTube's algorithm buried them.

Plus: it's fun to start the day with curated content insights instead of doom-scrolling a recommendation feed.

## What It Does

- Fetches the latest videos from a list of your favorite channels
- Extracts transcripts directly with **yt-dlp** (no external API, no credits, works offline)
- Summarizes key insights per video in Korean (or any language)
- Delivers a daily digest on demand or via cron

## Requirements

- [yt-dlp](https://github.com/yt-dlp/yt-dlp) installed and on PATH
- OpenClaw with `terminal` MCP enabled

No API keys, no third-party services, no per-transcript costs.

## How It Works

### Transcript Extraction (yt-dlp)

For each video, run via the `terminal` MCP:

```bash
cd ~/.openclaw/workspace/.cache/yt
yt-dlp --write-auto-sub --write-sub \
  --sub-lang "ko-orig,ko,en,en-orig" \
  --skip-download --sub-format vtt \
  -o "%(id)s.%(ext)s" \
  "https://youtu.be/<VIDEO_ID>"
```

This downloads the subtitle file as `<videoId>.<lang>.vtt`. No video is downloaded — only the subtitle track.

**Language priority:** `ko-orig` → `ko` → `en-orig` → `en`

### VTT Cleanup

VTT files contain timestamps and duplicate lines. Strip them before summarizing:

```python
import re

def clean_vtt(text):
    out, prev = [], ""
    for line in text.splitlines():
        if line.startswith("WEBVTT") or "-->" in line:
            continue
        if re.match(r"^(align|position|line|size):", line):
            continue
        line = re.sub(r"<\d{2}:\d{2}:\d{2}\.\d+>", "", line)
        line = re.sub(r"</?c>", "", line).strip()
        if not line or line == prev:
            continue
        out.append(line)
        prev = line
    return "\n".join(out)
```

### Metadata

Enrich each video with `mcp__youtube__getVideoDetails` (title, channel, duration, view count, description).

## How to Set It Up

### Option 1: Channel-based digest

Save your channel list to `~/.openclaw/workspace/assets/yt-channels.json`:

```json
[
  { "id": "UCxxxxxx", "name": "채널이름", "lang": "ko" },
  { "id": "UCyyyyyy", "name": "ChannelName", "lang": "en" }
]
```

Then prompt OpenClaw:

```text
매일 아침 8시에 yt-channels.json의 채널 목록에서 지난 24시간 이내 업로드된 영상을 가져와줘.
각 영상마다:
1. yt-dlp로 자막 추출 (.cache/yt/ 에 저장)
2. VTT 정제 후 핵심 내용 3~5줄 요약
3. 제목, 채널명, 링크 포함
결과를 memory/yt-digest-YYYY-MM-DD.md 에 저장하고 텔레그램으로 전송해줘.
```

### Option 2: Keyword-based digest

Track new videos about a specific topic:

```text
매일 "AI agents" 또는 "Claude Code" 키워드로 YouTube 검색을 해줘.
~/.openclaw/workspace/assets/seen-videos.txt 에 이미 처리한 영상 ID를 관리해서 중복 처리하지 마.
새 영상마다:
1. yt-dlp로 자막 추출
2. 핵심 3줄 요약
3. 내 작업과 관련성 메모

매일 오전 9시에 실행해줘.
```

## Tips

- `mcp__youtube__getChannelTopVideos` — 채널 최신 영상 목록 조회 (무료)
- `mcp__youtube__searchVideos` — 키워드 검색 (무료)
- yt-dlp VTT 파일은 `.cache/yt/` 에 캐시되므로 재실행 시 재다운로드 없음
- 7일 이상 된 VTT 파일은 주기적으로 정리 권장
- 자막 없는 영상은 스킵하고 스킵 이유를 다이제스트에 명시

## Important: Transcript Extraction Rule

`mcp__youtube__getTranscripts` 가 빈 배열을 반환해도 **"자막 없음"으로 판정하지 말 것**.  
항상 yt-dlp를 먼저 실행하고, `.vtt` 파일이 실제로 없을 때만 사용자에게 알린다.

## Related

- This already exists as a product — [Recapio - Daily YouTube Recap](https://recapio.com/features/daily-recaps)