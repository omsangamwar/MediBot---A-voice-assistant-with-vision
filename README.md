# AI Skin Specialist — Voice-to-Voice Edition

An AI-powered, voice-to-voice skin consultation assistant. A patient speaks a description of their concern and uploads a skin **image or video**; the app transcribes the voice, analyzes the visual, generates a doctor-style response, and speaks that response back — a full voice-in / voice-out loop, not just text.

> Inspired by [`AIwithhassan/ai-skin-specialist`](https://github.com/AIwithhassan/ai-skin-specialist), extended with a dedicated speech-to-text stage, video support, and a pluggable voice provider layer (Groq / ElevenLabs / Deepgram).

**⚠️ Disclaimer:** This project provides general informational guidance only. It is **not** a medical diagnosis and does not replace care from a licensed dermatologist or clinician. For severe symptoms, rapid spreading, fever, pain, bleeding, signs of infection, or any urgent concern, contact a licensed clinician immediately.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Technical Architecture](#technical-architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Setup](#setup)
- [Environment Variables](#environment-variables)
- [Run the App](#run-the-app)
- [How It Works](#how-it-works)
- [Roadmap](#roadmap)
- [Security & Privacy](#security--privacy)
- [Credits](#credits)
- [License](#license)

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

![Technical architecture diagram showing four phases: patient voice and image capture, speech-to-text, vision model and LLM response, and text-to-speech](docs/architecture.svg)

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
ai-skin-specialist/
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

### Installing FFmpeg & PortAudio

<details>
<summary>macOS</summary>

```bash
brew update
brew install ffmpeg portaudio
ffmpeg -version
```
</details>

<details>
<summary>Windows</summary>

```powershell
choco install ffmpeg portaudio
# or
scoop install ffmpeg portaudio
```
Restart your terminal after install. If `pyaudio` fails to build, install the [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/) and re-run `uv sync`.
</details>

<details>
<summary>Linux</summary>

Ubuntu/Debian:
```bash
sudo apt update
sudo apt install ffmpeg portaudio19-dev python3-dev build-essential
```
Fedora:
```bash
sudo dnf install ffmpeg portaudio-devel python3-devel gcc gcc-c++ make
```
Arch:
```bash
sudo pacman -S ffmpeg portaudio base-devel
```
</details>

## Setup

Install `uv` if you don't have it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh   # macOS/Linux
# or on Windows (PowerShell):
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Clone the project and sync dependencies:

```bash
git clone <your-repo-url> ai-skin-specialist
cd ai-skin-specialist
uv sync
```

`uv sync` reads `pyproject.toml` and `uv.lock`, creates `.venv`, installs Python 3.11 if needed, and installs all locked dependencies.

## Environment Variables

Copy the template and fill in your keys:

```bash
cp sample.env .env
```

```bash
GROQ_API_KEY=your_groq_api_key_here

# Choose at least one STT provider and one TTS provider
DEEPGRAM_API_KEY=your_deepgram_api_key_here
ELEVENLABS_API_KEY=your_elevenlabs_api_key_here

# Optional model overrides
WHISPER_MODEL=whisper-large-v3
GROQ_MODEL=meta-llama/llama-4-scout-17b-16e-instruct
DEEPGRAM_TTS_MODEL=aura-2-thalia-en
ELEVENLABS_VOICE_ID=your_preferred_voice_id
STT_PROVIDER=groq        # groq | elevenlabs
TTS_PROVIDER=deepgram    # deepgram | elevenlabs
```

Never commit a real `.env` file — keep API keys out of version control.

## Run the App

```bash
uv run python main.py
```

Gradio prints a local URL, typically:

```
http://127.0.0.1:7860
```

Open it in your browser.

## How It Works

1. Record (or upload) a patient voice description of the skin concern.
2. Upload a skin **image**, or a short **video** — a representative frame is extracted automatically for analysis.
3. Click **Analyze Concern**.
4. The app transcribes the audio (Phase 3), analyzes the image/frame + transcript with the vision model (Phase 1), converts the response to speech (Phase 2), and plays it back (Phase 4).
5. Review the transcript, the written doctor-style guidance, and listen to the generated audio response.

## Roadmap

- [ ] Native multi-frame video analysis (instead of a single extracted frame)
- [ ] Session history so a patient can ask follow-up questions with prior context
- [ ] Streaming TTS playback instead of waiting for the full audio file
- [ ] Optional local/offline STT and TTS fallback for privacy-sensitive deployments
- [ ] Structured output (severity flag, suggested next step) alongside the free-text response

## Security & Privacy

- Keep `.env` private and out of version control.
- Treat patient images, videos, and audio as sensitive data — avoid persisting them beyond what's needed for a single session.
- Define a clear data retention and privacy policy before any production or clinical-facing use.
- This is a research/demo project, not a HIPAA-compliant or clinically validated system.

## Credits

- Original concept and base implementation: [`AIwithhassan/ai-skin-specialist`](https://github.com/AIwithhassan/ai-skin-specialist)
- AI providers: [Groq](https://groq.com), [Deepgram](https://deepgram.com), [ElevenLabs](https://elevenlabs.io)
- Web framework: [Gradio](https://www.gradio.app)

## Medical Disclaimer

This app is an AI assistant for general skin-care information only. It cannot diagnose disease, prescribe medication, or replace a medical professional. For severe symptoms, rapid spreading, fever, pain, bleeding, signs of infection, or any urgent concern, contact a licensed clinician immediately.

## License

Add your preferred license (e.g. MIT) here.
