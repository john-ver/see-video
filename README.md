# see-video

**Your AI agent can now understand video. Inject video frames directly into context — no proxy model, no description handoff.**

---

## The Problem with "Describe and Pretend"

Most video pipelines for LLMs follow the same pattern: send the video to a
capable multimodal service (Gemini, GPT-4o), receive a text description back,
then pass that description to your main LLM. It works — but it's not the same
as your LLM watching the video. The model you're talking to never sees a single
frame. It reads a summary written by a different model, filtered through that
model's priorities, blind spots, and abstraction choices. The gap between
"a model described it" and "my LLM saw it" matters most when the video carries
nuance, motion, or detail that survives frames but dies in prose.

---

## What This Skill Does Differently

Instead of outsourcing perception, `see-video` extracts frames directly from
the video and injects them as an image sequence into the same context window
your LLM is already using. No proxy model. No description handoff. The frames
sit alongside the conversation — your LLM sees them the same way it sees
anything else you send. When it responds, it's drawing on its own visual
understanding of the content, not a secondhand account of one.

---

## The Core Library

This skill is built on [`llm-frames`](https://github.com/john-ver/llm-frames),
a small library designed specifically for LLM context injection — not human
viewing. Tools like `vcsi` produce contact sheets meant for people to browse;
`llm-frames` produces frame arrays optimized for model context windows:
LLM-friendly sizing, minimal token overhead, timestamp labeling, and (in Step 2)
synchronized audio transcript interleaving. It wraps `ffmpeg` under the hood
and exposes a clean interface the skill calls without worrying about format,
resolution tradeoffs, or merge logic. You can also use `llm-frames` directly
in your own pipeline outside of this skill.

---

## Step 1: Visual Context

The skill uniformly samples N frames across the video's duration and injects
them as a labeled image sequence into the LLM context. Each frame is tagged
with its timestamp so the model has a clear sense of temporal position.

**What the LLM receives:**

```
[00:02] <image>
[00:05] <image>
[00:09] <image>
[00:13] <image>
```

**Usage (in an OpenClaw agent):**

```bash
# sample 8 frames, inject into current context
bash command:"see-video /path/to/video.mp4 --frames 8"
```

No external API beyond your existing LLM. No description intermediary.
Just frames in context, ready to reason over.

---

## Step 2: Full Context — Visual + Audio

Step 2 adds audio. The skill extracts the audio track, transcribes it
(local Whisper or OpenAI API), then merges the transcript with the frame
sequence by timestamp — giving the model a unified timeline of what it
would have seen *and* heard at each moment.

**What the LLM receives:**

```
[00:02] <image>
[00:03] transcript: "I've been waiting..."
[00:05] <image>
[00:06] transcript: "...for you."
[00:09] <image>
[00:11] transcript: "don't go."
```

**Usage:**

```bash
# step 2: frames + audio transcript
bash command:"see-video /path/to/video.mp4 --frames 8 --audio"
```

The result is a complete, temporally coherent context block.
Visual, spoken, and sequential — all in one pass, all in the same window.

---

## Why Not Just Use Gemini (or any native video API)?

Gemini's native video API is excellent, and for many workflows it's the
right choice — especially if you don't mind routing media through a
third-party service and your main LLM isn't the one doing the video
reasoning anyway.

This skill is for when:

- Your primary model is **Claude (Bedrock), GPT-4o**, or any vision-capable
  LLM without a native video API
- You want **your** model — the one holding the conversation — to be the
  one doing the seeing
- You need video understanding to happen **in the same context window**
  as the ongoing conversation, not handed off to a separate service
- You prefer not to route media through a third-party video API

If Gemini is already your main model and its native video API covers your
use case, use that. This skill exists for everyone else.

---

## Sampling Modes

`see-video` supports two frame sampling strategies. The right choice depends
on your hardware, video content, and how much context budget you can afford.

---

### Uniform (default)

Divides the video into N equal intervals and extracts one frame per interval.
Fast, predictable, and works on any hardware. The simplest way to give your LLM
a temporal cross-section of the video.

```
[00:02] <image>   ← interval 1
[00:05] <image>   ← interval 2
[00:09] <image>   ← interval 3
[00:13] <image>   ← interval 4
```

Best for: short clips, static or slow-moving content, low-resource environments,
or when you want deterministic output regardless of what's in the video.

```bash
see-video video.mp4 --mode uniform --frames 8
```

---

### Highlight (scene-aware)

Analyzes the video for scene changes and motion intensity, then selects frames
at moments of maximum visual significance — cuts, peaks of movement, or
abrupt transitions. Instead of sampling at fixed intervals, it finds the frames
that actually carry information.

```
[00:01] <image>   ← scene cut detected
[00:04] <image>   ← motion peak
[00:07] <image>   ← scene cut detected
[00:11] <image>   ← motion peak
```

Best for: action-heavy content, multi-scene videos, or any case where uniform
sampling might land on near-identical or low-information frames.

> ⚠️ **Hardware note:** Highlight mode runs scene detection and motion analysis
> via `ffmpeg` filters, which is CPU-intensive. On low-resource machines this
> may be slow. If highlight mode fails or times out, `see-video` automatically
> falls back to uniform mode and logs a warning — output is never silently dropped.

```bash
see-video video.mp4 --mode highlight --frames 8
```

---

### Configuration

| Option | Default | Description |
|---|---|---|
| `--mode` | `uniform` | Sampling strategy: `uniform` or `highlight` |
| `--frames` | `8` | Number of frames to extract and inject |
| `--max-tokens` | — | Cap total image tokens; reduces `--frames` automatically if exceeded |
| `--fallback` | `true` | Fall back to `uniform` if `highlight` fails or times out |
| `--fallback-timeout` | `30` | Seconds before highlight analysis times out and fallback triggers |
| `--size` | `auto` | Output frame resolution: `auto`, `small`, `medium`, or `WxH` |
| `--audio` | `false` | Enable Step 2 audio transcript |

---

## Requirements

| Dependency | Step 1 | Step 2 |
|---|---|---|
| `ffmpeg` | ✅ required | ✅ required |
| Vision-capable LLM | ✅ required | ✅ required |
| `llm-frames` library | ✅ required | ✅ required |
| Whisper CLI or OpenAI API key | — | ✅ required |

---

## Installation

```bash
# Via ClawHub
npx clawhub@latest install see-video

# Manual
git clone https://github.com/john-ver/see-video
```

---

## SKILL.md — `description` field

```
Use when the user sends a video file and visual understanding is needed.
Samples frames from the video using llm-frames and injects them as a
timestamped image sequence directly into the current LLM context window —
no proxy model, no external description API. Step 2 adds audio transcript
interleaved with frames by timestamp for full audiovisual context. Requires
ffmpeg. Prefer this over Gemini-describe workflows when the primary LLM must
reason over the video itself, not a summary of it.
```
