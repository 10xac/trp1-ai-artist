# AI-Content Package: Codebase Exploration

> **Part 2 Deliverable** — TRP1 AI Content Generation Challenge

## 1. Package Structure

### Main Modules in `src/ai_content/`

| Module | Purpose |
|--------|---------|
| **`cli/`** | Command-line interface (Typer-based). Entry point: `ai-content` |
| **`config/`** | Configuration loading from `.env` and YAML. Pydantic settings. |
| **`core/`** | Foundational abstractions: Provider protocols, registry, result types, exceptions, job tracker |
| **`providers/`** | Provider implementations (Lyria, MiniMax, Veo, Kling, Imagen) |
| **`presets/`** | Pre-configured music and video style templates |
| **`pipelines/`** | Orchestration workflows (music, video, full music-video) |
| **`integrations/`** | Media processing (FFmpeg), YouTube upload, Archive.org |
| **`utils/`** | Lyrics parser, file handlers, retry logic |

### Provider Organization

Providers are organized by **API source** (not by content type):

```
providers/
├── google/          # Google Gemini APIs
│   ├── lyria.py     # Music (instrumental)
│   ├── veo.py       # Video (text-to-video, image-to-video)
│   └── imagen.py    # Images
├── aimlapi/         # AIMLAPI proxy
│   ├── client.py    # HTTP client for AIMLAPI
│   └── minimax.py   # Music with vocals (MiniMax 2.0)
└── kling/           # KlingAI direct
    └── direct.py    # Video (highest quality, slow)
```

**Registration**: Providers self-register via decorators (`@ProviderRegistry.register_music("lyria")`). Importing the provider module triggers registration. No central manifest file.

### Pipelines Directory Purpose

The `pipelines/` directory provides **orchestrated workflows** that combine multiple steps:

| Pipeline | Purpose |
|----------|---------|
| **`base.py`** | `PipelineResult`, `PipelineConfig` — shared abstractions for multi-step workflows |
| **`music.py`** | `MusicPipeline` — performance-first, lyrics-first, reference-based, provider comparison |
| **`video.py`** | `VideoPipeline` — text-to-video, image-to-video, provider comparison |
| **`full.py`** | `FullContentPipeline` — end-to-end: music → image → video → merge → optional upload |

Pipelines are used by **example scripts** and **programmatic access**, not directly by the CLI. The CLI calls providers directly.

---

## 2. Provider Capabilities

### Music Providers

| Provider | API | Vocals/Lyrics | Reference Audio | Realtime | Notes |
|----------|-----|---------------|-----------------|----------|-------|
| **lyria** | Google Gemini | ❌ Instrumental only | ❌ | ✅ | WebSocket streaming, BPM/temperature control, fastest |
| **minimax** | AIMLAPI (MiniMax 2.0) | ✅ Yes | ✅ Style transfer | ❌ | Async poll, lyrics with `[Verse]`/`[Chorus]` tags, non-English support |

**Key difference**: Use **Lyria** for instrumental (jazz, ambient, electronic). Use **MiniMax** for songs with vocals — provide lyrics via `--lyrics path/to/file.txt`.

### Video Providers

| Provider | API | Text-to-Video | Image-to-Video | Aspect Ratios | Speed |
|----------|-----|---------------|----------------|---------------|-------|
| **veo** | Google Gemini | ✅ | ✅ | 16:9, 9:16, 1:1 | ~30 seconds |
| **kling** | KlingAI Direct | ✅ | ✅ | 16:9, 9:16, 1:1 | 5–14 minutes |

**Image-to-video**: Both support `first_frame_url` / `--image`. The CLI currently expects an image **URL** for video (not a local path) — local images would need to be uploaded first.

**Vocals/Lyrics**: Only **MiniMax** (via AIMLAPI) supports vocals. Lyria explicitly ignores lyrics.

---

## 3. Preset System

### Music Presets (BPM & Mood)

| Preset | BPM | Mood | Tags |
|--------|-----|------|------|
| jazz | 95 | nostalgic | smooth, fusion, sophisticated |
| blues | 72 | soulful | delta, raw, authentic |
| ethiopian-jazz | 85 | mystical | ethio-jazz, modal, african |
| cinematic | 100 | epic | orchestral, film-score, triumphant |
| electronic | 128 | euphoric | house, edm, festival |
| ambient | 60 | peaceful | ambient, meditative, eno |
| lofi | 85 | relaxed | lofi, chill, study |
| rnb | 90 | sultry | rnb, neo-soul, modern |
| salsa | 180 | fiery | salsa, latin, cuban |
| bachata | 130 | romantic | bachata, latin, dominican |
| kizomba | 95 | sensual | kizomba, zouk, african |

### Video Presets (Aspect Ratio)

| Preset | Aspect Ratio | Duration | Style Keywords |
|--------|--------------|----------|----------------|
| nature | 16:9 | 5s | documentary, wildlife, golden-hour |
| urban | 21:9 | 5s | cyberpunk, urban, neon |
| space | 16:9 | 5s | sci-fi, space, contemplative |
| abstract | 1:1 | 5s | abstract, commercial, satisfying |
| ocean | 16:9 | 5s | ocean, underwater, paradise |
| fantasy | 21:9 | 5s | fantasy, dragon, epic |
| portrait | 9:16 | 5s | portrait, fashion, beauty |

### Adding a New Preset

**Music preset** (`src/ai_content/presets/music.py`):

```python
NEW_STYLE = MusicPreset(
    name="afrobeats",
    prompt="""[Afrobeats] [Talking drums, Highlife guitar]...""",
    bpm=105,
    mood="energetic",
    tags=["afrobeats", "highlife", "african"],
)
# Add to MUSIC_PRESETS dict
MUSIC_PRESETS["afrobeats"] = NEW_STYLE  # or include in list comprehension
```

**Video preset** (`src/ai_content/presets/video.py`):

```python
NEW_VIDEO = VideoPreset(
    name="sunset",
    prompt="Golden sunset over mountains...",
    aspect_ratio="16:9",
    duration=5,
    style_keywords=["sunset", "landscape", "golden-hour"],
)
# Add to VIDEO_PRESETS dict
```

---

## 4. CLI Commands

### Available Commands

| Command | Purpose |
|---------|---------|
| `ai-content music` | Generate music |
| `ai-content video` | Generate video |
| `ai-content list-providers` | List music, video, image providers |
| `ai-content list-presets` | List music and video presets |
| `ai-content music-status <id>` | Check MiniMax job status, optionally download |
| `ai-content jobs` | List tracked generation jobs |
| `ai-content jobs-stats` | Job statistics summary |
| `ai-content jobs-sync` | Sync pending jobs with API |

### Music Command Options

| Option | Default | Description |
|--------|---------|-------------|
| `--prompt`, `-p` | (required) | Music style prompt |
| `--provider` | lyria | `lyria` or `minimax` |
| `--style`, `-s` | None | Preset name (overrides prompt & BPM) |
| `--duration`, `-d` | 30 | Duration in seconds |
| `--bpm` | 120 | Beats per minute |
| `--lyrics`, `-l` | None | Path to lyrics file (MiniMax only) |
| `--reference-url`, `-r` | None | Reference audio URL (MiniMax only) |
| `--output`, `-o` | None | Output file path |
| `--force`, `-f` | False | Force generation even if duplicate exists |

**Note**: When using `--style`, `--prompt` is optional (the preset's prompt is used). You can run: `uv run ai-content music --style jazz --provider lyria --duration 30`

### Video Command Options

| Option | Default | Description |
|--------|---------|-------------|
| `--prompt`, `-p` | (required) | Scene description |
| `--provider` | veo | `veo` or `kling` |
| `--style`, `-s` | None | Preset name (overrides prompt & aspect) |
| `--aspect`, `-a` | 16:9 | Aspect ratio |
| `--duration`, `-d` | 5 | Duration in seconds |
| `--image`, `-i` | None | First frame image (URL required for now) |
| `--output`, `-o` | None | Output file path |

---

## 5. Architecture Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI (Typer)                              │
└────────────────────────────────┬────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                     ProviderRegistry                             │
│  (Decorator-based registration, lazy singleton instances)        │
└────────────────────────────────┬────────────────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
┌───────▼───────┐    ┌───────────▼──────────┐    ┌───────▼───────┐
│ MusicProvider │    │   VideoProvider      │    │ ImageProvider │
│ lyria, minimax│    │   veo, kling         │    │   imagen      │
└───────────────┘    └─────────────────────┘    └───────────────┘
        │                        │                        │
        └────────────────────────┼────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  External APIs          │
                    │  Google, AIMLAPI, Kling │
                    └────────────────────────┘
```

**Key design patterns**:
- **Protocol-based interfaces** — Structural subtyping, no inheritance required
- **Registry pattern** — Self-registration via decorators
- **Async-first** — All I/O is async
- **Result objects** — `GenerationResult` instead of exceptions for flow control

---

## 6. Quick Reference: Common Commands

```bash
# List options
uv run ai-content list-providers
uv run ai-content list-presets

# Music (instrumental)
uv run ai-content music --prompt "smooth jazz" --provider lyria --duration 30
uv run ai-content music --prompt "jazz" --style jazz --provider lyria --duration 30

# Music with vocals (MiniMax)
uv run ai-content music --prompt "lo-fi hip-hop" --provider minimax --lyrics lyrics.txt

# Video
uv run ai-content video --prompt "Lion in savanna" --provider veo --aspect 16:9
uv run ai-content video --prompt "nature" --style nature --provider veo --duration 5
```
