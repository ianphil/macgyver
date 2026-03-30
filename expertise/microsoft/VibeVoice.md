Here's the spec:

---

# VibeVoice — Product Spec

*microsoft/VibeVoice · analyzed March 2026*

---

## Problem Statement

Transcribing a one-hour meeting used to mean chunking it into thirty-second clips, stitching outputs back together, and then doing a second pass to figure out who said what. And synthesizing speech in real time — with a voice that actually sounds like a person, available within a quarter second of receiving text — was basically a research problem, not something you could deploy.

VibeVoice is Microsoft's answer to both of those problems. It's a family of open-source voice AI models — one that transcribes long-form audio in a single pass with speaker attribution baked in, and one that generates speech fast enough to feel live. A third model for high-fidelity multi-speaker synthesis exists but has been pulled from public distribution due to misuse concerns (more on that later).

The core bet: treat audio the same way LLMs treat text — represent it as tokens, process it with a large language model, and let the model figure out structure, speakers, and meaning jointly rather than in separate pipeline stages.

---

## Actors & Goals

**Developers** are the primary users. They're building applications — transcription services, voice assistants, podcast processing tools, accessibility tools — and need a programmatic interface. They want clean output, predictable behavior, and something they can tune for their domain without retraining from scratch.

**Operators** running production services need horizontal scale and an API contract they can rely on. They care about latency, throughput, GPU memory budgets, and whether this thing speaks OpenAI's API language so they don't have to rewire their clients.

**Researchers** want to fine-tune on domain-specific audio, extend the model architecture, and understand what's actually happening inside. They care about access to the model internals and a clear path from labeled data to adapted model.

**End users** — the humans whose voice gets transcribed, or who hear synthesized speech — care about accuracy, attribution, and the assurance that AI-generated audio is disclosed as such.

---

## Operator Value

Before VibeVoice, getting all three of these in one pass — transcription, speaker identity, and word-level timestamps — required a pipeline of separate models. You'd run Whisper (or something like it) for the words, a diarization model for the speakers, and then a fragile alignment step to match them up. Each seam was a failure point.

VibeVoice-ASR collapses that pipeline into a single model call. Hand it an hour of audio, get back a JSON document with speaker-attributed, timestamped segments. Done.

On the synthesis side: if you need speech that starts playing before you've finished generating the sentence — the kind of latency that makes a voice assistant feel responsive rather than robotic — VibeVoice-Realtime delivers the first audible audio within about 200–300 milliseconds, on hardware as modest as a consumer laptop or a cloud T4.

The vLLM serving plugin means you can scale this to production traffic without forking vLLM or writing custom serving code. Install the package, point vLLM at the model, get an OpenAI-compatible endpoint.

---

## Core Capabilities

### VibeVoice-ASR (7B)

- **Long-form transcription**: Processes up to 60 continuous minutes of audio in a single pass — not chunked, genuinely long-context.
- **Speaker diarization**: Identifies individual speakers throughout the recording and tags each segment with a speaker ID. No second-pass alignment needed.
- **Word-level timestamps**: Every segment includes start and end times in seconds.
- **50+ language support**: Handles multilingual recordings, including mid-conversation language switches, without configuration.
- **Hotword/context injection**: Operators can supply domain-specific terms (product names, jargon, proper nouns) at inference time to improve accuracy for specialized content. No retraining required.
- **Fine-tuning**: The ASR model can be adapted to specific domains using parameter-efficient training on labeled audio. Training data format is a paired audio + JSON file per recording.

### VibeVoice-Realtime (0.5B)

- **Streaming text-to-speech**: Accepts text incrementally and begins producing audio before the full input is available.
- **Sub-300ms first audio**: First audible chunk arrives within roughly 200–300 milliseconds on modern hardware.
- **Pre-built voice library**: Ships with 11 named English voices and experimental support for 9 additional languages (German, French, Italian, Japanese, Korean, Dutch, Polish, Portuguese, Spanish) with 2 voices each.
- **Long-form synthesis**: Can generate up to approximately 10 minutes of continuous audio in a single session.

### VibeVoice-TTS (1.5B) — restricted

- Model weights are publicly available on HuggingFace. Repository code has been removed as of September 2025 due to misuse concerns.
- When accessible, it supported: up to 90 minutes of multi-speaker audio generation, up to 4 distinct speakers with natural turn-taking, English and Chinese, and cross-lingual synthesis.
- Emergent spontaneous singing was documented as an unintentional but real capability.

---

## Observable Behaviors

### Transcribing a file

**Trigger**: User provides one or more audio files and a model path via CLI or Python API. Optionally includes a comma-separated list of hotwords.

**Response**: The model processes the audio and returns a structured JSON document containing a list of segments. Each segment includes speaker ID (integer), transcribed text, start time, and end time.

**Persistent effect**: Output is written to disk (CLI mode) or returned as a Python object (API mode). No state is retained between calls.

**Failure mode**: If the audio contains heavily overlapping speech, transcription accuracy degrades. Recordings with significant background music or noise may produce unreliable output. Codes, formulas, and special symbols are not handled reliably.

---

### Generating speech from text

**Trigger**: User provides a text file or string and a speaker name. A WebSocket client sends text to the server endpoint.

**Response**: Audio begins streaming within ~200–300ms. The system produces a 24kHz WAV stream. Generation metrics (real-time factor, token counts) are logged when running in verbose mode.

**Persistent effect**: Output audio saved to disk (file mode) or streamed to connected clients (WebSocket mode). Real-time factor below 1.0 indicates the system is generating faster than playback speed.

**Failure mode**: Very short inputs (3 words or fewer) may produce unstable audio. Non-English languages are marked experimental and results vary. Custom voice cloning is not supported — only the pre-bundled voice profiles work.

---

### Scaling to production

**Trigger**: Operator installs the package and launches the vLLM serving script, specifying data-parallel and/or tensor-parallel configuration.

**Response**: A server starts on the configured port, exposing an OpenAI-compatible chat completions endpoint (`/v1/chat/completions`). Nginx load-balances across replicas in data-parallel mode.

**Persistent effect**: Audio inputs submitted through the API are transcribed and results returned as streaming responses. No audio is persisted server-side.

**Failure mode**: Memory pressure on the GPU can cause out-of-memory errors; operators can reduce memory utilization settings or increase GPU count. For very long recordings, the server can enter a repetition loop — a recovery script exists to detect and break out of this state.

---

### Fine-tuning on domain data

**Trigger**: Developer provides labeled audio (paired MP3 + JSON annotation files) and launches a distributed training job across multiple GPUs.

**Response**: Training runs for the configured number of epochs, producing a set of adapter weights. Progress is tracked via a compatible experiment tracking tool.

**Persistent effect**: Fine-tuned adapter weights saved to an output directory. These can be loaded alongside the base model at inference time to improve accuracy on the target domain without replacing the base model.

**Failure mode**: Batch size of 1 per GPU is the documented setting — this is a memory-constrained workload. Gradient checkpointing is required for most hardware configurations.

---

## Edge Cases

- **Language switching mid-recording**: The ASR model handles this natively. A recording that starts in English and switches to French mid-sentence will be transcribed without configuration changes.
- **Video files as input**: Audio is extracted automatically from MP4 and WebM containers. You don't need to strip the audio track first.
- **Extremely short synthesis inputs**: Inputs of three words or fewer may produce artifacts or unstable audio. Longer inputs are more reliable.
- **Long audio near the 60-minute limit**: Transcription accuracy may degrade near the context boundary. The model has been tested on recordings up to 60 minutes; beyond that is uncharted.
- **Chinese synthesis**: Documented as occasionally unstable. The larger TTS variant (which is restricted) showed better results; the available models are less reliable for Chinese.
- **Background music appearing in synthesized audio**: This is a documented quirk of the TTS model — something in the training data occasionally surfaces as unexpected background music in generated audio. It's not configurable and not fully predictable.

---

## Non-Functional Constraints

- **GPU required for inference.** A single NVIDIA T4 (16 GB VRAM) is the stated minimum for the ASR and Realtime models. The 7B ASR model consumes roughly 20 GB in half-precision; multi-GPU serving is required for memory-constrained environments.
- **Apple Silicon supported for Realtime.** The 0.5B streaming model has been verified at real-time speed on an Apple M4 Pro. The larger ASR model is not tested on Apple hardware.
- **FFmpeg must be installed.** Audio decoding depends on FFmpeg being available on the system path. Without it, audio files cannot be processed.
- **Python 3.9 or later.** Older Python versions are not supported.
- **Flash Attention 2 recommended.** The models use standard attention as fallback, but Flash Attention 2 is the recommended path for production performance. Requires NVIDIA GPU with compute capability 7.0 or higher.
- **Streaming TTS context ceiling.** The realtime model has an 8,000-token context window, which corresponds to roughly 10 minutes of generated audio. Longer sessions require a new context.
- **ASR is not real-time.** The 7B ASR model processes at roughly 7–12× slower than real-time on a single T4. A 60-minute recording takes 10–20 minutes to transcribe. This is a batch workload, not a live transcription system.

---

## Non-Goals

- **Live meeting transcription.** VibeVoice-ASR is designed for recorded audio, not live streaming. Latency is measured in minutes per hour of audio, not seconds.
- **Custom voice cloning.** The Realtime model ships with a fixed library of voice profiles. There is no mechanism to enroll a new voice from a sample recording.
- **Overlapping speech.** The diarization model expects speakers to take turns. Simultaneous speech from multiple participants is a known limitation.
- **Text normalization.** The model does not post-process transcribed text to normalize numbers, dates, abbreviations, or punctuation. What the model produces is what you get.
- **CPU-only inference.** There is no supported CPU inference path. A GPU is required.
- **On-device / edge deployment.** The smallest model (0.5B) can run on consumer-grade GPU hardware, but there is no quantized, mobile, or edge-optimized build.

---

## What's Actually Impressive

A few things stood out while going through this:

**The 60-minute single-pass thing is genuinely unusual.** Most speech recognition systems — commercial and open-source — chunk audio before processing it. The chunking introduces seams: repeated words at boundaries, lost context, split sentences. VibeVoice-ASR avoids all of that by treating the entire recording as a single context window. An hour of audio becomes one model call that understands the whole conversation.

**Joint diarization + transcription + timestamps.** Getting all three from one model call — knowing who said what and when — without a separate alignment step is the kind of thing that sounds obvious once you see it but wasn't the norm. The output format is clean: a list of segments, each one complete.

**The vLLM plugin architecture is thoughtful.** Zero changes to vLLM source code. The package registers itself as an entry point, vLLM picks it up automatically on install, and you get a production-grade OpenAI-compatible endpoint. That's a real engineering convenience for teams that already have vLLM in their stack.

**200ms TTS latency on a T4.** That's genuinely real-time. The 0.5B streaming model manages to feel live because it *is* live — first audio within a quarter second, sustained generation at or below real-time factor.

**The TTS model can apparently sing.** Not because anyone trained it to — it emerged from whatever music was in the training data. It's not reliable or configurable, but it's there. Make of that what you will.

---

## Responsible AI Note

The full TTS model (1.5B) is the elephant in the room. Microsoft pulled the repository code in September 2025 citing misuse — the model produces high-quality synthetic speech convincing enough that it was being used for impersonation. The model weights remain available, but without the serving code.

This is worth naming plainly: voice synthesis at this quality level is a dual-use capability. The project documentation asks users to disclose AI-generated audio and comply with local laws. That's the right thing to say. Whether it's sufficient is a harder question.

Security vulnerabilities should be reported to Microsoft directly — not as public GitHub issues.

---

## Suspected Implementation Leakage

*Statements that are probably true but describe mechanism rather than promise — belong in a technical spec.*

- The tokenizer operates at 7.5 Hz frame rate, achieving 320× compression for ASR and 3,200× for streaming TTS. This is a remarkable engineering choice and worth knowing for researchers, but it's a *how*, not a *what*.
- The language model backbone is derived from Qwen2.5. This explains certain behavioral characteristics (multilingual capability, code-switching) but the product promise holds regardless of which base model is used.
- The diffusion head uses DPM-Solver++ with a configurable number of sampling steps (5 for streaming, 20+ for high quality). The tradeoff between steps and quality is user-visible, but the scheduler name is not.
- Data parallel mode uses nginx for load balancing across model replicas. Users care that horizontal scaling works and that the endpoint stays consistent; the load balancer choice is implementation detail.