# Rethinkr Companion — README

**Privacy-first, on-device wearable thinking partner **

> A pocket-sized compute unit + wired headset that listens, transcribes, indexes, and reasons — a private external memory and conversational assistant that runs locally on an Edge GPU devkit (Jetson Orin Nano-class). 

---

## Table of contents

1. Vision & position
2. MVP scope (Tier C)
3. What success looks like (MVP metrics)
4. Hardware BOM (Tier C)
5. High-level architecture
6. Memory system & data model
7. Software stack (recommended)
8. Quick start — flash & run
9. Example commands & snippets
10. Privacy, security & legal guidance
11. Power, battery & thermal notes
12. Roadmap & milestones (90–120 days)
13. Testing plan & metrics
14. Demo script + video storyboard
15. Funding ask & budget (Blueprint-ready)
16. Contributing
17. License

---

## 1. Vision & position

Rethinkr Companion is a personal thinking partner that turns ephemeral speech into durable, useful memory and reasoning. It is *local-first*, permission-driven, and designed to amplify human reflection and focus — not to surveil. The Tier C prototype proves the feasibility of **real-time on-device retrieval-augmented reasoning** using an Edge GPU devkit.

This README focuses on an engineering-ready MVP that emphasises privacy, latency, and high-quality on-device reasoning.

---

## 2. MVP scope (Tier C — Edge GPU / Jetson Orin Nano)

**Goal:** a working prototype that can capture meeting/lecture audio, transcribe locally with high quality, perform speaker diarization, index semantic embeddings on-device, and run a quantized LLM for fast summarization & Q&A.

**Included features (MVP):**

* Pocket compute unit (Jetson Orin Nano Super Developer Kit or equivalent)
* Wired headset with high-SNR MEMS microphone
* Local VAD + wake-word support
* Local STT (whisper.cpp or optimized on-device model)
* Speaker diarization + alignment (whisperX / pyannote pipeline)
* On-device embeddings + HNSW index for recall
* Local quantized LLM (llama.cpp / GGML-quantized) for RAG-style summarization and short-form Q&A
* Mobile/desktop control app (React Native / Electron) for privacy controls, search, and export

**Out-of-scope (MVP):** continuous cloud streaming by default, full ASR accuracy on noisy phone-call channels, long-term cloud-based backup (opt-in only)

---

## 3. What success looks like (MVP metrics)

* STT word accuracy ≥ 85% in quiet-to-moderate environments for English (measured on 30–60 min test sessions).
* Summaries useful (user study): average helpfulness ≥ 4/5 across 20 tests.
* Query latency (local retrieval + short LLM answer) ≤ 2s for simple recall queries.
* Battery life: ≥ 6 hours capture in intermittent duty cycle mode (VAD + wake-word).
* Privacy baseline: local-first by default; encrypted local storage; explicit user opt-in for backups.

---

## 4. Hardware BOM (Tier C — recommended)

**Core compute & peripherals**

* NVIDIA Jetson Orin Nano Super Developer Kit (or equivalent Edge GPU dev kit) — *recommended*. (Dev kit, power supply).
* High-SNR wired earphones with inline MEMS microphone (or external lavalier MEMS mic) — low-latency, high-SNR recommended.
* USB audio interface (if improving mic quality / analog mic) — optional.
* 3D-printed or CNC’d pocket enclosure for Jetson + battery.

**Power & storage**

* High-capacity battery pack for Jetson (rated for 20W+ peak if mobile) OR bench PSU for lab testing.
* M.2 NVMe SSD (for fast local model load + audio storage).

**Optional (stretch)**

* Edge TPU / Coral accelerator (if offloading certain operations) — optional.
* External USB Wi-Fi/BT antenna (if improved connectivity is needed).

**Estimate (prototype-level, approximate):**

* Jetson Orin Nano dev kit: ~$249–499 (depending on vendor / kit)
* M.2 NVMe 250GB: $30–60
* High-SNR mic/headset: $30–80
* Battery pack + power hardware: $50–150
* Enclosure + misc: $30–80
  **Prototype total:** ~$400–900 (ask $1,000 in funding to cover parts + shipping + extras + contingency)

> For the Blueprint submission request, request around **$1,000** to cover devkit, batteries, storage, prototyping, and small cloud credits for optional heavy jobs.

---

## 5. High-level architecture

```
[Wired Headset / Mic] -> [Pocket Compute Unit (Jetson)]
    -> VAD / Wake-word
    -> Local STT (whisper.cpp)
    -> Diarization & Alignment (whisperX / pyannote)
    -> Sentence segmentation -> Embeddings -> HNSW index (on-disk)
    -> RAG agent (llama.cpp quantized) for summarization/Q&A
    -> App UI (React Native / Electron) for search, privacy controls, export

Optional: encrypted cloud backup (user opt-in)
```

**Notes:** keep audio files and raw transcripts encrypted at rest. Only condensed semantic notes are retained for fast recall by default.

---

## 6. Memory system & data model

Design principles: salience-driven retention, user control, explainable retrieval.

**Data layers:**

* Raw audio files (chunked, short) — encrypted, archived (optional by user).
* Transcripts with timestamps + speaker labels — stored locally (SQLite + attachments).
* Semantic units (condensed notes): {id, title, one_line_summary, tags, embedding, source_refs, salience_score, timestamp}

**Retention policy:**

* New episodes live in short-term store for 7–14 days.
* Weekly condensation job: low-salience episodes condensed to 1–2 lines and archived.
* High-salience items (tagged/saved) stay in semantic index indefinitely unless user chooses to delete.

**Salience scoring:** `salience = f(repetition, direct user-save, speaker-weight, named-entity-density, recency)`

**Retrieval:** combine vector similarity (embedding) + recency boost + metadata filters (speaker, tag).

---

## 7. Software stack (recommended)

* OS & runtime: Ubuntu 22.04 (Jetson-optimized image) or JetPack recommended for Jetson dev kit.
* Containerization: Docker + NVIDIA Container Toolkit (recommended for reproducibility).

**Core components:**

* VAD / wake-word: `webrtcvad` and/or Picovoice/Porcupine for wake-word.
* Offline STT: `whisper.cpp` (quantized GGML models) or a Jetson-optimized model.
* Diarization / alignment: `whisperX` + `pyannote.audio` (for best speaker labeling).
* Embeddings: small sentence encoder (e.g., `sentence-transformers` distilled model, quantized) or local transformer encoder converted to GGML.
* Vector DB: `hnswlib` (C++/Python) on local storage.
* LLM runtime: `llama.cpp` (GGML quantized models) or `ggml` variants for on-device inference.
* App: React Native (mobile) or Electron (desktop) for the control plane.
* Storage: SQLite for metadata + encrypted blob store for audio/transcripts.

**Notes on models:** choose quantized GGML variants for CPU/GPU-friendly inference. Keep largest LLM size small enough to fit onboard memory for interactive reasoning (7B-class quantized model is a practical balance for many Edge GPU dev kits).

---

## 8. Quick start — flash, build, run (developer flow)

> Assumption: Jetson Orin Nano dev kit with Ubuntu / JetPack configured and NVIDIA Docker available.

### 1) Provision the dev kit

* Install JetPack and NVIDIA Container Toolkit (follow NVIDIA Jetson docs).
* Set hostname, create a `rethinkr` user, and enable SSH for remote development.

### 2) Clone repo (project root contains Dockerfiles)

```bash
git clone https://github.com/your-org/rethinkr-companion.git
cd rethinkr-companion
```

### 3) Build containers (example)

```bash
# Build the inference container with required libs (whisper.cpp, llama.cpp, hnswlib, pyannote)
./scripts/build_inference_container.sh
```

### 4) Attach audio device and test capture

```bash
# Example: record 30s to test.wav
arecord -D hw:0,0 -f cd -t wav -d 30 test.wav
```

### 5) Run STT on audio with whisper.cpp (container)

```bash
docker run --gpus all -v $PWD:/workspace -it rethinkr/inference:latest /bin/bash
# inside container
./whisper.cpp/main -m models/ggml-medium.bin -f /workspace/test.wav -otxt -t 4
```

### 6) Start the pipeline (dev script)

```bash
# starts capture -> stt -> diarization -> embedding -> index
./scripts/run_pipeline_dev.sh --device hw:0 --model-dir ./models
```

**Tip:** run the pipeline first in short sessions (1–5 min) to validate the flow and tune VAD/wake-word thresholds.

---

## 9. Example commands & snippets

**Whisper.cpp quick inference**

```bash
# from whisper.cpp repo (example)
./main -m models/ggml-large.bin -f audio_chunk.wav -of transcribed.txt -t 6
```

**Llama.cpp sample prompt (local)**

```bash
# Interactive generate with quantized model
./main -m models/ggml-model-q4_0.bin -p "Summarize the following context: <CONTEXT>" -n 256
```

**Python snippet: create and query HNSW index with hnswlib**

```python
import hnswlib
import numpy as np

dim = 1536  # embedding dim
p = hnswlib.Index(space='cosine', dim=dim)
p.init_index(max_elements=100000, ef_construction=200, M=16)

# add
embeddings = np.random.rand(1000, dim).astype('float32')
p.add_items(embeddings, list(range(1000)))

# query
labels, distances = p.knn_query(np.random.rand(1, dim).astype('float32'), k=5)
print(labels, distances)
```

**Example summarization prompt (RAG loop)**

```
You are a private assistant. Using the following context snippets retrieved from the user's recent meeting transcripts, produce a concise 3-4 sentence summary that highlights decisions, action items, and open questions. Below the summary list clear action items as bullets.

Context:
- <snippet 1>
- <snippet 2>
- ...
```

---

## 10. Privacy, security & legal guidance

* **Default local-first mode.** Audio, transcripts, embeddings, and models run locally. Cloud sync available only by user opt-in.
* **Encryption at rest.** Use libsodium or similar to encrypt audio blobs and transcripts using a key that the user controls.
* **Explicit consent UI.** UI must present a simple consent flow and require explicit acknowledgement before capturing others.
* **Visual recording indicator.** Hardware LED visible when the device is listening or recording.
* **Logging & deletion.** Provide easy deletion of specific recordings and a full-data wipe option.
* **Legal compliance.** Educate users about local wiretapping/recording laws via the app and surfacing reminders before multi-party recording.

---

## 11. Power, battery & thermal notes

* Jetson dev kits are power-hungry; for mobile prototypes, ensure a battery solution that can deliver 20W+ peaks (or accept tethered power for long sessions).
* Thermal throttling is possible under heavy inference load; design vents, or active fans and consider lowering model size or batching jobs to manage heat.
* Duty-cycle capture (VAD + wake-word) dramatically reduces total energy consumption vs continuous raw audio capture.

---

## 12. Roadmap & milestones (90–120 days)

**Phase 0 — Research & infra (1–2 weeks)**

* Acquire devkit, set up JetPack, test container infra.
* Smoke test whisper.cpp and llama.cpp on the devkit.

**Phase 1 — Capture & STT pipeline (3–4 weeks)**

* Implement VAD, chunking, whisper.cpp integration, and local transcripts.
* Deliverable: sample 30–60 min transcripts with timestamps.

**Phase 2 — Diarization & embedding pipeline (3–4 weeks)**

* Integrate whisperX + pyannote for speaker attribution; implement embedding encoder and HNSW index.
* Deliverable: searchable index + UI demo for query.

**Phase 3 — RAG assistant & memory condensation (4–6 weeks)**

* Run local quantized LLM for summarization and Q&A; implement memory condensation job and salience scoring.
* Deliverable: demo performing “summarize last meeting” and answering recall queries.

**Phase 4 — UX, enclosure, testing (2–4 weeks)**

* Build enclosure, battery integration, muting switch, privacy flows.
* Deliverable: Blueprint-ready prototype + 60s demo video.

**Phase 5 — User testing & polish (2–4 weeks)**

* 10–20 testers, iterate, and finalize submission materials.

---

## 13. Testing plan & metrics

* **Unit tests** for audio capture, STT pipeline, diarization alignment, embedding generation, and indexing.
* **Integration tests** for the full RAG pipeline (capture → query → LLM answer).
* **User testing**: recruit 10–20 testers to record typical sessions (lectures, meetings) and collect subjective ratings for transcript accuracy, summary usefulness, and privacy comfort.
* **Automated benchmarks**: measure inference latency, memory usage, and power draw for representative tasks.

---

## 14. Demo script + storyboard (60s)

1. 0–5s: Title card — “Rethinkr Companion — Private wearable thinking partner (Tier C MVP)”
2. 5–15s: Product shot — headset + pocket compute unit; LED indicator on.
3. 15–30s: Live demo — user asks: “Summarize my last meeting” → show 3-line summary on app.
4. 30–45s: Retrieval demo — ask: “What decision did we make about X?” → show snippet with timestamp and playback option.
5. 45–60s: Close — one-line vision + funding ask (parts + dev time).

---

## 15. Funding ask & budget (Blueprint-ready)

**Request:** $1,000 (one-time)

**Budget breakdown (example):**

* Jetson Orin Nano dev kit: $399
* NVMe 512GB: $50
* High-SNR mic/headset: $50
* Battery + power hardware: $150
* Enclosure & prototyping: $100
* Misc (SD cards, USB adapters, shipping): $100
* Contingency / cloud credits / testing participants: $151

**Total:** $1,000

---

## 16. Contributing

1. Fork the repo.
2. Create feature branches per `CONTRIBUTING.md`.
3. Use provided Docker containers for local dev.
4. Open PRs with tests and a short design note.

---

## 17. License

MIT — see `LICENSE`.

---

## Appendix A — Useful resources & commands (dev notes)

* Build whisper.cpp (refer to whisper.cpp README) and put models under `models/ggml-<size>.bin`.
* Build llama.cpp with CUDA/CUBLAS support where available on Jetson for faster inference.
* Use `hnswlib` (Python) for embedding indices; store serialized index on NVMe.

**Sample development script**

```bash
# Run dev container (example)
docker run --gpus all -v $PWD:/workspace -it rethinkr/inference:dev /bin/bash
# inside container
./scripts/dev_capture_and_index.sh --device hw:0 --out ./data
```

---

### Final note

This README is intentionally pragmatic: it aims to get you from concept → working Tier C prototype as quickly as possible while protecting user privacy and showing technical credibility. The Tier C path is the fastest way to demonstrate on-device RAG reasoning and to make a compelling case (both technically and narratively) in a Blueprint or similar grant application.

If you want, I will:

* generate the `Dockerfile`, `scripts/build_inference_container.sh`, and `scripts/run_pipeline_dev.sh` next;
* create a minimal `app/` control-plane skeleton (React Native or Electron) with privacy screens;
* or convert this README into a `README.md` file in a GitHub repo and populate a starter commit (with basic scripts & Dockerfiles).

Choose the next step and I will produce it right away.
