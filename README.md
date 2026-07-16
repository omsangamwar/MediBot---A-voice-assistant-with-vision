# SkinSense AI — A voice-assistant-with-vision.

An AI-powered, voice-to-voice skin consultation assistant. A patient speaks a description of their concern and uploads a skin **image or video**; the app transcribes the voice, analyzes the visual, generates a doctor-style response, and speaks that response back — a full voice-in / voice-out loop, not just text
---

## Overview

Traditional "upload a photo, get an AI opinion" skin-check tools stop at text. This project closes the loop: the patient **talks**, the assistant **sees and listens**, and the assistant **talks back** — like a real consultation, minus the ability to touch or examine physically (hence the disclaimer).

The app is built around four cooperating phases (see the architecture diagram below), each of which can be swapped for a different provider without touching the others:

1. **Capture** — record the patient's voice and accept an image or video upload (Gradio UI).
2. **Understand the voice** — transcribe the patient's spoken concern into text (Speech-to-Text).
3. **Understand the skin** — send the transcribed query + the image/video frame to a vision-capable LLM and get back a doctor-style response (Vision + LLM).
4. **Speak back** — convert that response into natural speech and play it to the patient (Text-to-Speech).

## Features

- 🎙️ **Voice-to-voice consultation** — speak your concern, hear the answer back, not just read it.
- 🖼️ **Image + 🎥 video input** — upload a photo for direct analysis, or a short video (a representative frame is extracted and analyzed alongside the transcript).
- 🧠 **Vision-grounded LLM reasoning** — the model reasons over both what it sees and what the patient said, not just a generic text prompt.
- 🔌 **Pluggable AI providers** — Groq, ElevenLabs, and Deepgram are all supported for STT/TTS; swap providers via environment variables without changing app logic.
- 🌐 **Simple Gradio web UI** — no separate frontend to build or deploy; runs locally with one command.
- 🔒 **Privacy-conscious by design** — no data persistence beyond the session by default; see [Security & Privacy](#security--privacy).

## Technical Architecture
<img width="1316" height="722" alt="image" src="https://github.com/user-attachments/assets/d3d214ab-21b8-4168-8971-c437f5c4ffb8" />

### Phase 1 — Vision Model & LLM Response
The uploaded image (or an extracted video frame) is sent — together with the transcribed patient query — to a **vision-capable LLM (Groq)**. The model returns a structured, doctor-style written response: possible explanations, general care guidance, and red flags that warrant seeing a real clinician.

### Phase 2 — Text-to-Speech (TTS)
The LLM's written response is converted into a spoken audio file using **Deepgram** or **ElevenLabs** text-to-speech, so the patient receives an actual spoken reply instead of only reading text.

### Phase 3 — Speech-to-Text (STT)
The patient's recorded voice note is transcribed into text using **Groq Whisper** or **ElevenLabs** speech-to-text, producing the "transcribed text / user query" that feeds Phase 1.

### Phase 4 — Gradio Web Interface
The patient-facing layer: a microphone recorder, an image/video upload control, and playback of the generated doctor audio response — all wired together in a single Gradio app.

**End-to-end flow:**

```
Patient speaks  ──▶  Audio Recorder ──▶  Speech-to-Text  ──▶  Transcribed Text/Query
                                                                        │
Patient uploads image/video ───────────────────────────────────────────┼──▶  Vision Model
                                                                        │            │
                                                                        │            ▼
                                                                        │      LLM Response
                                                                        │            │
                                                                        ▼            ▼
                                                        Doctor reply  ◀── Audio file ◀── Text-to-Speech
```

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Web interface | **Gradio** | Microphone recording, image/video upload, chat-style result display |
| Speech-to-Text | **Groq Whisper** (`whisper-large-v3`) / **ElevenLabs STT** | Transcribes the patient's spoken concern |
| Vision + reasoning | **Groq vision model** (e.g. `meta-llama/llama-4-scout-17b-16e-instruct`) | Analyzes the image/video frame + transcript, drafts the doctor-style response |
| Text-to-Speech | **Deepgram Aura** / **ElevenLabs TTS** | Converts the written response into spoken audio |
| Language & tooling | **Python 3.11+**, **uv** | Dependency, virtual-env, and lockfile management |
| Supporting libs | `pydub`, `pyaudio`/`sounddevice`, `speechrecognition`, `pillow`, `python-dotenv`, `numpy` | Audio capture/conversion, image preprocessing, env config |

## Project Structure

```text
skinsense-ai/
|-- main.py                       # Gradio application entry point (wires all 4 phases together)
|-- voice_of_the_patient.py       # Microphone recording + STT (Groq / ElevenLabs)
|-- brain_of_the_doctor_groq.py   # Groq vision/text response generation (default path)
|-- brain_of_the_doctor.py        # Alternate provider-agnostic vision/text implementation
|-- voice_of_the_doctor.py        # TTS audio generation (Deepgram / ElevenLabs)
|-- video_frame_extractor.py      # Pulls a representative frame from uploaded video for vision analysis
|-- sample.env                    # Environment variable template
|-- pyproject.toml                # Python package metadata and dependencies
|-- uv.lock                       # Locked dependency versions generated by uv
|-- .python-version               # Python version pinned for uv
|-- docs/
|   `-- architecture.svg          # Technical architecture diagram (this README)
`-- README.md
```

## Requirements

- Python 3.11 or newer
- [`uv`](https://github.com/astral-sh/uv) package manager
- **FFmpeg** — audio/video conversion, used by `pydub` and frame extraction
- **PortAudio** — microphone support, used by `pyaudio`/`speechrecognition`
- API keys: **Groq**, and at least one of **Deepgram** or **ElevenLabs**
