# Genesis — Android Embedded AI Assistant

> A fully on-device AI second brain for Android. Records your voice, transcribes it with Whisper, extracts structured memory with Gemma 3, and lets you query everything through a local chat interface — zero cloud, zero latency, zero data leaving your phone.

<p align="center">
  <img src="preview.png" width="30%" alt="Genesis app preview" />
  &nbsp;&nbsp;
  <img src="gallery.jpg" width="30%" alt="Screenshot 1" />
  &nbsp;&nbsp;
  <img src="gallery2.jpg" width="30%" alt="Screenshot 2" />
</p>

---

## What It Does

Genesis runs two neural networks locally on your phone and connects them into a closed-loop cognitive pipeline:

1. **Record** — tap to capture audio via a persistent foreground service (survives lock screen, background, and Android's aggressive process killing)
2. **Transcribe** — Whisper converts speech to text entirely on-device using `whisper.rn`
3. **Extract** — Gemma 3 (1B GGUF) reads the transcript and writes structured entries — notes, todos, and named facts — into a local memory store
4. **Chat** — a local assistant powered by the same Gemma model with full memory injected as context

---

## Why It's Non-Trivial

### On-Device Inference Pipeline

Running two separate neural networks (Whisper ASR + Gemma 3 LLM) sequentially on mobile hardware requires careful memory and threading management. Both models are GGUF-quantized and loaded into native runtimes (`whisper.rn` / `llama.rn`) that communicate back to JS via the React Native bridge. The pipeline is:

```
Microphone PCM → WAV encode → 16kHz resample → Whisper inference → transcript text
                                                                         ↓
                                                      Gemma 3 structured extraction
                                                                         ↓
                                                             Memory store (append-only)
```

### Android 14 Foreground Service

Android 14 kills `shortService` foreground services after ~3 minutes. Genesis uses a `microphone`-typed persistent foreground service via Notifee to hold the audio session open indefinitely. The recording state is managed outside the React component tree so it survives tab navigation and screen lock.

### Selective Memory Extraction

The LLM doesn't run on every word — it triggers after a file transcription completes, or every ~400 characters of live transcript, with a final pass on recording stop. This keeps battery and thermal impact low while still catching insights as they happen. The model is prompted to return structured JSON and silently skips extraction if nothing noteworthy is detected.

### Memory as Shared Singleton

All tabs share a single module-level memory store. Additions from the Record tab are immediately visible in the Chat tab with no prop drilling, no context API, and no re-renders on unrelated components.

### Live Audio Visualization

The animated "blob" on the Record screen is a spline-interpolated SVG path driven by React Native Reanimated worklets running on the UI thread. Each of the 8 control points follows:

```
P_i(t) = R · (sin(ωt + φᵢ) · scale + 1)
```

where `scale` increases with recording amplitude, giving organic visual feedback without dropping frames.

---

## Architecture

```
app/
  (tabs)/
    index.tsx        # Record tab — live + file transcription, memory extraction trigger
    chat.tsx         # Chat tab — memory-augmented local LLM chat
    memory.tsx       # Memory tab — structured view of notes, todos, facts
    models.tsx       # Model manager — download/load Whisper + Gemma models

services/
  whisper-service.ts     # Whisper.rn wrapper, WAV encoding, 16kHz resampling
  llm-service.ts         # Gemma 3 inference, memory extraction, chat completion
  memory-store.ts        # Append-only in-memory store with subscriber notifications
  background-service.ts  # Notifee foreground service lifecycle management
  native-runtime.ts      # Capability detection for native inference modules
```

---

## Stack

| Layer | Technology |
|---|---|
| Framework | React Native + Expo (managed → bare workflow) |
| Speech-to-Text | `whisper.rn` (native GGUF, no cloud) |
| Language Model | `llama.rn` with Gemma 3 1B GGUF |
| Background audio | `@notifee/react-native` foreground service |
| Animation | React Native Reanimated 4 (UI thread worklets) |
| Graphics | `react-native-svg` animated path |
| File picking | `expo-document-picker` |
| Navigation | Expo Router v6 |
| Language | TypeScript |

---

## Getting Started

### Prerequisites

- Node 18+
- Android Studio + Android SDK (API 34+)
- A physical Android device (emulators can't run GGUF inference at usable speed)

### Setup

```bash
git clone https://github.com/amr-radwan1/genesis2026.git
cd genesis2026/gen2026

npm install
npx expo prebuild --clean -p android
npx expo run:android
```

Or use the dev script (Windows):

```powershell
npm run android:devclient
```

That script mirrors the project to `C:\g26` to avoid Windows path-length build failures, runs prebuild, builds and installs the APK, and starts the Expo dev client on port 8083.

### Models

Download from the in-app **Models** tab or manually place GGUF files on device storage:
- **Whisper**: `ggml-base.en.bin` (recommended)
- **Gemma**: `gemma-3-1b-it-q4_k_m.gguf`

---

## Limitations

- Android only (iOS foreground audio service API differs significantly)
- Memory resets on cold start — AsyncStorage persistence is a planned addition
- Gemma 3 1B is fast but context-limited; longer conversations may lose early memory
- Model files (~600MB Whisper + ~800MB Gemma Q4) must be downloaded separately

---

## Contributing

PRs welcome. Focus areas: AsyncStorage persistence, iOS support, streaming Whisper output directly into LLM context in real time.
