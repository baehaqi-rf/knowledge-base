---
id: "kb-ai-video-editor-remotion-pipeline-v1"
title: "Autonomous AI Video Editing Architecture with Claude/Antigravity, Remotion, FFmpeg & Whisper"
version: "1.0.0"
status: "production-standard"
domain: "ai-agents-workflows"
stack:
  - "Remotion v4 (@remotion/media, @remotion/captions, @remotion/transitions, @remotion/effects)"
  - "FFmpeg & FFprobe (npx remotion ffmpeg, loudnorm, silencedetect)"
  - "Faster-Whisper (CTranslate2) & @remotion/whisper-webgpu"
  - "yt-dlp"
  - "HyperFrames / HTML-in-Canvas"
  - "React / TypeScript / Python"
triggers:
  - "ai video editor claude code remotion"
  - "automated video editing ffmpeg faster-whisper yt-dlp"
  - "programmatic reels tiktok shorts captions remotion"
  - "style.md reference video extraction workflow"
  - "silence detection retake deduplication edl remotion"
  - "hyperframes remotion motion graphics pipeline"
related:
  - "kb-design-ios-liquid-glass-cross-platform-v1"
  - "kb-graphics-webgpu-liquid-glass-v2"
sources:
  - title: "Official Remotion Agent Skills Repository (remotion-dev/skills v4.0.538)"
    url: "https://github.com/remotion-dev/skills"
  - title: "Remotion Captions & TikTok-Style Word Paging (@remotion/captions)"
    url: "https://www.remotion.dev/docs/captions/"
  - title: "Remotion Local GPU Transcription (@remotion/whisper-webgpu)"
    url: "https://www.remotion.dev/docs/whisper-webgpu/"
  - title: "Faster-Whisper (SYSTRAN CTranslate2 Whisper Implementation)"
    url: "https://github.com/SYSTRAN/faster-whisper"
  - title: "Baseline Reference Reel: AI Editing Workflow with Claude Code (@byaico)"
    url: "https://www.instagram.com/reel/Dft_Hs-I0Lu/"
last_updated: "2026-11-22"
---

# Autonomous AI Video Editing Architecture with Claude/Antigravity, Remotion, FFmpeg & Whisper

**Scope:** Production engineering specification for autonomous, code-first video editing pipelines that transform raw talking-head or screen-recorded footage (`raw/`) into studio-grade short-form (9:16 Reels/TikTok/Shorts) and long-form (16:9) deliverables (`done/`) using reference style extraction (`references/` -> `style.md`), dual-engine word-level Whisper transcription, adaptive EBU R128 FFmpeg silence cutting, automatic retake deduplication (`edl.json`), and non-destructive shared-clock Remotion compositions.

## 1. Decision Scope

### Ideal Use Cases

- **Talking-Head + Programmatic Stage Reels/Shorts (9:16):** Educational tech/finance/product videos combining a bottom PiP facecam (`raw/take_001.mp4`), center kinetic captions, and top programmatic UI mockups, code cards, or animated charts.
- **Automated Rough-Cut & Retake Pruning:** Cleaning unscripted or multi-take recordings by stripping dead air, filler pauses, and stumbled retakes via word-timestamp Edit Decision Lists (`edl.json`) without manual timeline dragging.
- **Style-Locked Content Factories:** Replicating consistent pacing, 3-second hook mechanics, typography hierarchy, and color/audio grading across batches of videos governed by a single version-controlled `style.md`.
- **Hybrid Code + Motion Plate Compositing:** Pairing React/TypeScript Remotion timelines with `@remotion/effects`, `<HtmlInCanvas>`, or standalone HTML/CSS/GSAP graphic plates (HyperFrames pattern).

### Anti-Patterns (Do NOT Use When)

- **Blind Frame-by-Frame LLM Context Dumping:** NEVER extract hundreds of full-resolution PNG frames at high FPS from 3–5 reference videos into a text/code LLM prompt. It exhausts token budgets and fails to capture audio rhythm.
- **Physical MP4 Sub-Clip Fragmentation:** NEVER slice a raw recording into dozens of re-encoded `clip_01.mp4..clip_40.mp4` files on disk for standard jump cuts. Use Remotion's non-destructive `<TransitionSeries.Sequence trimBefore={...} durationInFrames={...}>` on the original source file.
- **Color-Graded Cinema / Multi-Cam Live Grading:** Complex optical-flow color grading, raw LOG/BRAW debayering, and manual multi-hour documentary storytelling belong in DaVinci Resolve or Premiere Pro.

## 2. Reference Baseline & Bottleneck Audit

### Baseline Workflow Audit (`@byaico` Claude Code Reel Pattern)

The viral 9-step Claude Code video editing workflow (`raw/`, `references/`, `done/` + 5 tools: Remotion, HyperFrames, FFmpeg, faster-whisper, `yt-dlp` + 3 prompts) establishes a strong modular mental model (separating style extraction into `style.md` from raw editing). However, naive execution of its three prompts exposes critical production bottlenecks—most notably demonstrated by a **duplicate/looped segment bug inside the reference reel itself**:

| Dimension | Naive / Prototype Pattern | Production Bottleneck | Production Standard (Improved) |
| :--- | :--- | :--- | :--- |
| **Reference Analysis (`Prompt 2`)** | *"Pelajari frame per frame: pacing, subtitle, warna, musik, 3 detik awal"* | Extracting every frame of 3–5 videos floods LLM context ($50\text{k}–200\text{k}+$ tokens) and still cannot hear beat/LUFS dynamics | **Telemetry-First Deconstruction:** `ffprobe` scene-cut detection + `loudnorm` LUFS + Whisper WPM + **Sparse Start/Mid/End Keyframes** (or native multimodal video API) |
| **Retake & Loop Handling (`Prompt 3`)** | Raw transcript piped directly into timeline without semantic deduplication | Speaker retakes or duplicated `<Sequence>` blocks render back-to-back in final export (causing the repetition bug seen in the baseline reel) | **Automated Retake Deduplication (`edl.json`):** Prefix + Jaccard/SequenceMatcher ($\ge 0.72$) pruning keeps only the final clean take |
| **Silence Detection** | Fixed hardcoded dB threshold (e.g., `-30dB`) or `/dev/null` shell calls | Clips quiet consonants on soft mics, misses room noise on loud mics, and crashes on Windows PowerShell (`/dev/null` missing) | **2-Pass Adaptive EBU R128:** Pass 1 `loudnorm` extracts `input_thresh`; Pass 2 `silencedetect` uses `input_thresh` with `-f null -` |
| **Timeline & Caption Sync** | Trimming `<Video>` separately from a global caption overlay | Shifting or retiming a single clip desynchronizes all downstream subtitles | **Shared-Clock `<TransitionSeries.Sequence>`:** `trimBefore`, `durationInFrames`, and `playbackRate` sit on the parent sequence wrapping both `<Video>` and inline `<BasicCaptions>` |
| **FFmpeg Provisioning** | Requires manual OS-wide FFmpeg installation and PATH configuration | Breaks on fresh machines or restricted environments | **Zero-Install Bundled Binary:** Invoke `npx remotion ffmpeg` and `npx remotion ffprobe` directly from Remotion's compositor |
| **style.md Contract** | Free-form prose summary written by LLM | Ambiguous adjectives (*"fast pacing, modern font"*) produce inconsistent layouts and UI safe-zone collisions | **Quantitative `style.md` Schema:** Explicit pixel safe zones, cut frequency (ASL), spring configs, hex tokens, and `-14 LUFS` audio target |

## 3. Production Architecture & Core Invariants

### System Dataflow Diagram

```text
+-----------------------------------------------------------------------------+
|                        WORKSPACE CONTRACT (<project-root>)                  |
|   raw/ (Source Takes)        references/ (yt-dlp)        done/ (Final MP4)  |
+-----------+---------------------------+--------------------------+----------+
            |                           |                          ^
            |                           v                          |
            |            +------------------------------+          |
            |            | PHASE A: Reference Telemetry |          |
            |            |  1. ffprobe scene-cut ASL    |          |
            |            |  2. loudnorm + Whisper WPM   |          |
            |            |  3. Sparse/Multimodal vision |          |
            |            +--------------+---------------+          |
            |                           |                          |
            |                           v                          |
            |                  [style.md Contract]                 |
            |             (Safe Zones, Springs, Tokens)            |
            |                           |                          |
            v                           |                          |
+-------------------------------+       |                          |
| PHASE B: Audio & EDL Engine   |       |                          |
|  1. 2-Pass EBU R128 Silence   |       |                          |
|     (loudnorm -> silencedetect)|      |                          |
|  2. Word-Level Whisper        |       |                          |
|     (faster-whisper / WebGPU) |       |                          |
|  3. Retake Deduplication      |       |                          |
|     (Similarity >= 0.72 prune)|       |                          |
+---------------+---------------+       |                          |
                |                       |                          |
                v                       v                          |
       [edl.json + captions.json]       |                          |
                |                       |                          |
                +-----------+-----------+                          |
                            v                                      |
        +-----------------------------------------------+          |
        | PHASE C: Remotion Shared-Clock Composition    |          |
        |  - <TransitionSeries.Sequence trimBefore=...> |          |
        |  - Top Stage: Programmatic UI / HyperFrames   |          |
        |  - Mid Stage: @remotion/captions Kinetic Text |          |
        |  - Bottom PiP: <Video> A-Roll Facecam         |          |
        +-----------------------+-----------------------+          |
                                |                                  |
                                v                                  |
        +-----------------------------------------------+          |
        | PHASE D: Pre-Render QA Gate                   |          |
        |  1. Zero-Overlap / Zero-Duplicate EDL Audit   |----------+
        |  2. Still-Frame Safe-Zone Inspection          |
        |  3. -14 LUFS Loudness & True-Peak Check       |
        +-----------------------------------------------+
```

### Pillar 1: Deterministic Workspace & Quantitative `style.md` Schema

Every project directory MUST maintain a strict separation between immutable inputs (`raw/`, `references/`), machine-readable intermediate artifacts (`style.md`, `edl.json`, `captions.json`), and verified outputs (`done/`).
- `style.md` MUST encode numeric parameters: canvas dimensions, platform UI occlusion margins, hook duration (`frames 0..90` at 30 FPS), cut interval bounds, spring physics (`{ mass, damping, stiffness }`), color tokens, and caption batching (`combineTokensWithinMilliseconds`).

### Pillar 2: Token-Efficient Reference Deconstruction

When extracting style from reference videos downloaded via `yt-dlp`:
1. **Structural Cut Telemetry:** Run `npx remotion ffmpeg -i ref.mp4 -vf "select=gt(scene\,0.25),showinfo" -f null -` to calculate exact cut timestamps and Average Shot Length ($\text{ASL} = T_{\text{total}} / N_{\text{cuts}}$).
2. **Sparse Visual Sampling:** Sample at most **3 frames per detected scene** (`startFrame + 1`, `midFrame`, `endFrame - 1`) scaled to `720p` width, OR analyze the MP4 once via a native video multimodal endpoint.
3. **Synthesis, Not Plagiarism:** Extract structural timing and typographic contrast rules while binding project-specific brand tokens.

### Pillar 3: Dual-Engine Transcription, Adaptive Silence Cutting & Retake Deduplication

#### A. 2-Pass Adaptive Silence Detection
Measure integrated loudness ($I$) and gating threshold ($T_{\text{gate}}$) via EBU R128 `loudnorm`, then set the silence floor dynamically:

$$T_{\text{silence}} = \mathrm{clamp}(T_{\text{gate}}, -42\text{ dB}, -24\text{ dB})$$

Any contiguous audio interval where amplitude remains below $T_{\text{silence}}$ for duration $d \ge 0.30\text{ s}$ is marked as a non-speech gap and stripped via sequence trimming (leaving a natural $80\text{ ms}$ pre-roll and $120\text{ ms}$ post-roll breath pad around spoken clauses).

#### B. Word-Level `Caption` Invariant
All transcription engines (`faster-whisper` or `@remotion/whisper-webgpu`) MUST emit arrays conforming to `@remotion/captions`:
- Timestamps (`startMs`, `endMs`, `timestampMs`) MUST stay relative to the **original unedited source media start (`0 ms`)**, never shifted to composition-global time.
- Every word after the first token MUST include a leading space (`" word"`) so `@remotion/captions` preserves whitespace accurately across line and page breaks.

#### C. Retake & Duplicate Take Pruning
Given consecutive spoken segments $S_i$ and $S_{i+1}$ with normalized token sequences $W_i, W_{i+1}$:

$$\mathrm{Sim}(S_i, S_{i+1}) = \frac{2 \cdot |M|}{|W_i| + |W_{i+1}|}$$

If $\mathrm{Sim}(S_i, S_{i+1}) \ge 0.72$ or the first 3 normalized words match identically ($W_i[0..2] = W_{i+1}[0..2]$) within a $15\text{ s}$ window, $S_i$ is classified as a failed/stumbled take and **discarded in favor of $S_{i+1}$**.

### Pillar 4: Non-Destructive Shared-Clock Remotion Timeline & Editorial 9:16 Stage

- **Shared Timing Boundary:** Place `trimBefore`, `durationInFrames`, and `playbackRate` on `<TransitionSeries.Sequence>` (or `<Sequence>`), wrapping `<Video>` (`@remotion/media`), `<BasicCaptions>`, and synchronized B-roll/HUD cards together.
- **Editorial 3-Zone Vertical Architecture (`1080x1920`):**
  1. **Top Visual Stage (`y: 120..1120px`):** Interactive UI mockups, terminal tables, step badges (`01 · App`), or `<HtmlInCanvas>` effects driven by `spring()` and `interpolate()`.
  2. **Center-Lower Caption Band (`y: 1140..1260px`):** High-contrast word-by-word or chunked kinetic captions (`createTikTokStyleCaptions` with `combineTokensWithinMilliseconds: 750`).
  3. **Bottom PiP Facecam (`y: 1280..1600px`, safe above `y: 1640px` Reels chrome):** Rounded A-roll container (`380x280px`, `borderRadius: 28px`) that scales smoothly between full/split-screen on the 3-second hook and compact PiP during technical walkthroughs.

## 4. Reference Implementation Patterns

### 4.1. Quantitative `style.md` Specification Contract

```markdown
---
style_id: "editorial-dark-gold-reels-v1"
canvas: { width: 1080, height: 1920, fps: 30 }
safe_zones_px: { top: 110, bottom: 280, left: 64, right: 140 }
audio_target: { speech_lufs: -14.0, bgm_lufs: -26.0, true_peak_dbtp: -1.0 }
---

# Editorial Dark-Gold Tech Reel Style Specification

## 1. Pacing & Cut Invariants
- **0–3s Hook:** Split-screen facecam + scrolling reference card carousel; mandatory cut or zoom impulse at `frame 45` (1.5s) and `frame 90` (3.0s).
- **Silence Trimming:** Strip pauses `>= 300ms`; retain `80ms` pre-roll and `120ms` post-roll padding.
- **Shot Length (ASL):** Re-anchor visual stage every `2.0s – 3.5s` (`60–105 frames`).

## 2. Visual & Typography Tokens
- **Palette:** Canvas `#0D0D0D`, Elevated Card `#161618`, Card Border `rgba(255,255,255,0.10)`, Primary Accent `#FACC15` (Gold), Alert `#EF4444`.
- **Stage Typography:** Step Pill (`#FACC15` fill, `#0D0D0D` bold sans `22px`), UI Body (`JetBrains Mono` `24px`).
- **Kinetic Captions:** Mixed Editorial Serif Italic (`Instrument Serif` / `Playfair Display`, `56px`) + Sans (`Inter`, `54px`); active emphasized keyword highlighted in `#FACC15`.
- **Motion Physics:** ` { mass: 0.8, damping: 18, stiffness: 210 } `; zero CSS transitions.
```

### 4.2. Cross-Platform Adaptive Silence Detection & Loudness Normalization (FFmpeg)

```powershell
# 1. Probe EBU R128 integrated loudness and adaptive gating threshold (Windows & POSIX safe: '-f null -')
npx remotion ffmpeg -i raw/take_001.mp4 -map 0:a -af loudnorm=print_format=json -f null -

# 2. Run adaptive silence detection using the measured threshold (e.g. -32dB) and 0.30s min duration
npx remotion ffmpeg -i raw/take_001.mp4 -map 0:a -af "silencedetect=noise=-32dB:d=0.30" -f null -

# 3. Optional pre-normalization of raw speech audio to -14 LUFS broadcast target
npx remotion ffmpeg -y -i raw/take_001.mp4 -c:v copy -af "loudnorm=I=-14:TP=-1.0:LRA=11" raw/take_001_norm.mp4
```

### 4.3. Faster-Whisper Word-Level `Caption[]` & Retake Deduplication (Python)

```python
import difflib
from faster_whisper import WhisperModel

def normalize_clause(text: str) -> str:
    return " ".join("".join(c.lower() for c in text if c.isalnum() or c.isspace()).split())

def is_stumbled_retake(prev_text: str, curr_text: str, threshold: float = 0.72) -> bool:
    a, b = normalize_clause(prev_text), normalize_clause(curr_text)
    if not a or not b:
        return False
    if difflib.SequenceMatcher(None, a, b).ratio() >= threshold:
        return True
    wa, wb = a.split(), b.split()
    return len(wa) >= 3 and len(wb) >= 3 and wa[:3] == wb[:3]

def extract_deduplicated_edl(video_path: str, fps: int = 30, model_size: str = "small") -> list[dict]:
    model = WhisperModel(model_size, device="auto", compute_type="default")
    segments, _ = model.transcribe(
        video_path,
        word_timestamps=True,
        vad_filter=True,
        vad_parameters={"min_silence_duration_ms": 300},
    )

    raw_clauses: list[dict] = []
    for seg in segments:
        words = []
        for idx, w in enumerate(seg.words or []):
            token = w.word if w.word.startswith(" ") else f" {w.word}"
            if not raw_clauses and idx == 0:
                token = token.lstrip()
            s_ms, e_ms = int(round(w.start * 1000)), int(round(w.end * 1000))
            words.append({
                "text": token,
                "startMs": s_ms,
                "endMs": e_ms,
                "timestampMs": (s_ms + e_ms) // 2,
                "confidence": round(float(w.probability), 4) if w.probability else None,
            })
        if words:
            raw_clauses.append({
                "text": seg.text.strip(),
                "startMs": words[0]["startMs"],
                "endMs": words[-1]["endMs"],
                "captions": words,
            })

    # Prune stumbled retakes: always keep the later clean take
    clean_clauses: list[dict] = []
    for clause in raw_clauses:
        if clean_clauses and is_stumbled_retake(clean_clauses[-1]["text"], clause["text"]):
            clean_clauses.pop()
        clean_clauses.append(clause)

    edl: list[dict] = []
    for i, c in enumerate(clean_clauses):
        pad_start = max(0, c["startMs"] - 80)
        pad_end = c["endMs"] + 120
        trim_before = int(round((pad_start / 1000.0) * fps))
        end_frame = int(round((pad_end / 1000.0) * fps))
        edl.append({
            "sequenceId": f"seq_{i + 1:02d}",
            "trimBefore": trim_before,
            "durationInFrames": max(1, end_frame - trim_before),
            "captions": c["captions"],
        })
    return edl
```

### 4.4. Non-Destructive Shared-Clock Editorial Reel Composition (Remotion TSX)

```tsx
import React, { useMemo } from "react";
import { AbsoluteFill, interpolate, spring, staticFile, useCurrentFrame, useVideoConfig } from "remotion";
import { Video } from "@remotion/media";
import { TransitionSeries } from "@remotion/transitions";
import { createTikTokStyleCaptions, type Caption } from "@remotion/captions";

interface EditorialSequenceProps {
  stepBadge: string;
  title: string;
  videoSrc: string;
  captions: Caption[];
  stageContent: React.ReactNode;
}

const EditorialShot: React.FC<EditorialSequenceProps> = ({
  stepBadge,
  title,
  videoSrc,
  captions,
  stageContent,
}) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  const cardEntrance = spring({
    frame,
    fps,
    config: { mass: 0.8, damping: 18, stiffness: 210 },
  });

  const { pages } = useMemo(
    () => createTikTokStyleCaptions({ captions, combineTokensWithinMilliseconds: 750 }),
    [captions]
  );

  const currentTimeMs = (frame / fps) * 1000;
  const activePage = pages.find(
    (p) => currentTimeMs >= p.startMs && currentTimeMs < p.startMs + p.durationMs
  );

  return (
    <AbsoluteFill style={{ backgroundColor: "#0D0D0D", color: "#F8FAFC" }}>
      {/* Zone 1: Top Programmatic Visual Stage */}
      <div
        style={{
          position: "absolute",
          top: 120,
          left: 64,
          right: 64,
          opacity: interpolate(cardEntrance, [0, 1], [0, 1], {
            extrapolateLeft: "clamp",
            extrapolateRight: "clamp",
          }),
          translate: `0px ${interpolate(cardEntrance, [0, 1], [28, 0], {
            extrapolateLeft: "clamp",
            extrapolateRight: "clamp",
          })}px`,
        }}
      >
        <div
          style={{
            display: "inline-block",
            backgroundColor: "#FACC15",
            color: "#0D0D0D",
            fontWeight: 700,
            fontSize: 22,
            padding: "6px 18px",
            borderRadius: 999,
            marginBottom: 24,
          }}
        >
          {stepBadge}
        </div>
        <div
          style={{
            backgroundColor: "#161618",
            border: "1px solid rgba(255, 255, 255, 0.10)",
            borderRadius: 24,
            padding: 36,
          }}
        >
          <div style={{ fontSize: 32, fontWeight: 700, marginBottom: 16 }}>{title}</div>
          {stageContent}
        </div>
      </div>

      {/* Zone 2: Mid-Lower Kinetic Caption Band */}
      {activePage ? (
        <div
          style={{
            position: "absolute",
            top: 1140,
            left: 80,
            right: 80,
            textAlign: "center",
            fontSize: 54,
            lineHeight: 1.2,
          }}
        >
          {activePage.tokens.map((t) => {
            const isActive = currentTimeMs >= t.fromMs && currentTimeMs < t.toMs;
            return (
              <span
                key={`${t.fromMs}-${t.text}`}
                style={{
                  color: isActive ? "#FACC15" : "#F8FAFC",
                  fontStyle: isActive ? "italic" : "normal",
                  fontWeight: isActive ? 700 : 500,
                }}
              >
                {t.text}
              </span>
            );
          })}
        </div>
      ) : null}

      {/* Zone 3: Bottom PiP Facecam (Synchronized with parent trimBefore) */}
      <div
        style={{
          position: "absolute",
          bottom: 300,
          left: 350,
          width: 380,
          height: 280,
          borderRadius: 28,
          overflow: "hidden",
          border: "1.5px solid rgba(255, 255, 255, 0.18)",
        }}
      >
        <Video
          src={staticFile(videoSrc)}
          style={{ width: "100%", height: "100%", objectFit: "cover" }}
        />
      </div>
    </AbsoluteFill>
  );
};

export const EditorialAiReel: React.FC = () => {
  const { fps } = useVideoConfig();

  return (
    <TransitionSeries name="Editorial AI Reel">
      <TransitionSeries.Sequence
        name="01 - App Setup"
        trimBefore={Math.round(3.92 * fps)}
        durationInFrames={Math.round(2.7 * fps)}
      >
        <EditorialShot
          stepBadge="01 · App"
          title="Claude Desktop · Code"
          videoSrc="raw/take_001.mp4"
          stageContent={<div>Environment: Lokal (~/Desktop/AI Editor)</div>}
          captions={[
            { text: "Step", startMs: 3920, endMs: 4300, timestampMs: 4110, confidence: 0.99 },
            { text: " pertama,", startMs: 4300, endMs: 4850, timestampMs: 4575, confidence: 0.99 },
            { text: " buka", startMs: 4850, endMs: 5200, timestampMs: 5025, confidence: 0.99 },
            { text: " aplikasinya.", startMs: 5200, endMs: 6500, timestampMs: 5850, confidence: 0.98 },
          ]}
        />
      </TransitionSeries.Sequence>
    </TransitionSeries>
  );
};
```

## 5. Performance Budgets & Verification Checklist

| Metric | Production Budget | Verification Method |
| :--- | :--- | :--- |
| **Reference Ingestion Token Cost** | $< 4\text{k tokens}$ per reference video | Telemetry + sparse scene keyframes (3 frames/scene max) |
| **Duplicate / Retake Sequence Count** | $0$ repeated clauses or overlapping `trimBefore` windows | Automated `edl.json` similarity & range overlap audit |
| **Audio Loudness & Peak** | $-14\text{ LUFS} \pm 1.0\text{ LU}$ / True Peak $\le -1.0\text{ dBTP}$ | `npx remotion ffmpeg -i done/out.mp4 -af loudnorm=print_format=json -f null -` |
| **Caption Sync Drift** | $< 33\text{ ms}$ ($\le 1\text{ frame}$ @ 30 FPS) | Shared parent `<TransitionSeries.Sequence>` clock check |
| **Safe-Zone Occlusion Violations** | $0\text{ px}$ overlap with platform UI chrome | Still-frame audit at hook, mid-step, and CTA frames |

### Runtime QA Verification Checklist

- [ ] **Anti-Repetition EDL Audit:** Verify `edl.json` and JSX `<TransitionSeries.Sequence>` blocks contain zero repeated takes or duplicated sequence blocks.
- [ ] **Shared-Clock Trim Test:** Confirm `trimBefore` and `durationInFrames` are placed on `<TransitionSeries.Sequence>` (not solely on `<Video>`), keeping captions and video locked when trims change.
- [ ] **Safe-Zone Still Inspection:** Render checkpoint frames (`npx remotion render <Id> .tmp_qa --frames=15,90,180`) and verify neither captions nor the PiP facecam fall inside the bottom `280px` or right `140px` Reels UI overlay zone.
- [ ] **Zero CSS Animation Audit:** Grep `src/` to confirm zero CSS `@keyframes`, `transition-*`, or Tailwind `animate-*` classes exist in Remotion components.
- [ ] **Windows/POSIX FFmpeg Null Sink Check:** Ensure all diagnostic FFmpeg commands use `-f null -` rather than OS-specific `/dev/null`.

## 6. Agent Guardrails (MUST / NEVER)

- **MUST:** Structure editing workspaces into `raw/`, `references/`, `done/`, `style.md`, and `edl.json`.
- **MUST:** Use `npx remotion ffmpeg` / `npx remotion ffprobe` with `-f null -` for portable zero-install media probing and 2-pass adaptive EBU R128 silence detection (`loudnorm` -> `silencedetect`).
- **MUST:** Deduplicate stumbled speaker retakes (similarity $\ge 0.72$ or identical 3-word prefix) before generating `<TransitionSeries.Sequence>` nodes.
- **MUST:** Keep `<Video>` and `<BasicCaptions>` / caption renderers inside the **same** `<TransitionSeries.Sequence>` timing boundary, preserving source-relative millisecond timestamps (`0 ms` = start of raw file) and leading spaces (`" word"`).
- **MUST:** Enforce quantitative `style.md` specifications (explicit safe-zone margins, ASL cut intervals, spring physics, and $-14\text{ LUFS}$ loudness).
- **NEVER:** Extract hundreds of raw video frames at high FPS into an LLM prompt to analyze reference videos.
- **NEVER:** Pre-slice raw footage into dozens of re-encoded physical MP4 clips on disk for normal timeline jump cuts.
- **NEVER:** Use CSS `transition`, CSS `@keyframes`, `requestAnimationFrame`, or `Math.random()` inside Remotion components.
- **NEVER:** Ship a rendered MP4 to `done/` without running the anti-repetition EDL check and multi-frame still audit.

## 7. Glossary & References

- **ASL (Average Shot Length):** Mean duration between consecutive visual cuts ($T_{\text{total}} / N_{\text{cuts}}$), governing video pacing.
- **EBU R128 (`loudnorm`):** Broadcast audio loudness specification measuring integrated loudness in LUFS and adaptive noise gating thresholds.
- **EDL (Edit Decision List):** Machine-readable JSON manifest (`edl.json`) mapping kept speech segments to non-destructive `trimBefore` and `durationInFrames` coordinates.
- **HyperFrames:** Open-source (Apache 2.0) HTML/CSS/GSAP/Three.js frame-capture motion graphics engine often paired with Remotion in agent workflows.
- **Retake Deduplication:** Algorithmic pruning of repeated/stumbled spoken clauses in raw recordings so only the final clean delivery enters the edit.
- **Shared-Clock Sequence:** Remotion pattern where `<TransitionSeries.Sequence>` owns `trimBefore` and `durationInFrames`, synchronizing child `<Video>`, `<Audio>`, and `<BasicCaptions>` layers on a single timeline clock.
