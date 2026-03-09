# Case Study: Audio Story Ingestion Pipeline

**Role:** Solution Architect · Tech Lead  
**Stack:** NestJS · BullMQ · FastAPI · faster-whisper · BGE-M3 · OpenAI GPT-4o · Google Imagen (Vertex AI) · Qdrant · MongoDB · Yandex S3 · Cloudflare Tunnel

---

## Background

MDS Collection is an audio platform for original spoken-word stories (~12K users). Audio content is opaque by default — without transcripts, tags, and semantic indexing, search doesn't work. The task: design and implement an async pipeline that takes a raw audio URL and produces a fully indexed story record: transcript, summary, tags, vector embeddings, and a cover image.

---

## Key Questions

**What does "done" mean?** Transcript stored, summary and tags attached, content searchable, cover image available. All four must complete — partial state must be recoverable without restarting from scratch.

**Where does ML inference run?** On a dedicated Mini PC (Intel N95, 16GB RAM). Self-hosting faster-whisper and BGE-M3 eliminates per-call API costs for the two heaviest steps. Cloudflare Tunnel exposes the ML service to the VDS without a static IP or open ports.

**Why Russian-specific transcription?** Managed APIs handle Russian adequately but struggle with narration cadence and proper nouns. Whisper medium gives meaningfully better results for this content type.

**What happens on failure?** Each step writes its result to MongoDB before completing. A retry picks up exactly where it left off — no re-downloading, no re-processing already-completed work.

---

## Back-of-Napkin Estimates

Assumptions: 20-min average story, 50 stories/day at peak.

| Component | Daily cost |
|---|---|
| faster-whisper + BGE-M3 (local) | $0 (hardware sunk) |
| OpenAI GPT-4o enrichment | ~$0.75 |
| Google Imagen cover generation | ~$1.00 |
| Yandex S3 storage growth | ~$0.50 |
| **Total** | **~$2.25/day (~$68/month)** |

Self-hosting Whisper cuts what would otherwise be ~$15/day at this volume.

---

## Tradeoffs

**BullMQ FlowProducer vs. chained async calls** — A simple `await` chain loses all progress on failure. FlowProducer models the pipeline as a dependency tree: each step runs only after the previous one completes, and retries independently. Essential when transcription alone takes 10+ minutes.

**Pipeline state in MongoDB vs. job data only** — BullMQ job data is ephemeral. Writing step results to a `pipeline` field on the story document gives idempotency (re-running checks if the step already ran) and a full audit trail in one place.

**Self-hosted Whisper vs. managed API** — Quality gap for Russian narration is real, cost gap is decisive. Risk is Mini PC availability — mitigated by BullMQ retry: jobs queue and drain automatically when the service recovers.

**Cloudflare Tunnel vs. VPN** — The Mini PC has no static IP. Tunnel gives stable HTTPS access, no open ports, auto-reconnects on IP change, runs as a systemd service. Simpler to operate than WireGuard.

**Two Qdrant collections vs. one** — Story-level discovery and passage-level RAG retrieval need different granularities. Separate collections for summaries and chunks keep query logic clean.

**ffmpeg pre-processing before Whisper** — User-uploaded audio varies in quality. An ffmpeg filter chain (noise reduction, bandpass, normalization → 16kHz mono) consistently improves transcript accuracy, which directly drives search quality.

---

## Architecture

### Infrastructure

**VDS** — NestJS API, BullMQ workers, Redis, MongoDB, Qdrant.  
**Mini PC** — FastAPI service: `/transcribe` (faster-whisper) and `/embed` (BGE-M3). Accessed over Cloudflare Tunnel with shared-secret auth.  
**Yandex Object Storage** — S3-compatible. Raw audio, enhanced audio, generated covers.

[Audio Processing Pipeline](./Audio-Processing-Pipeline.jpg)

### Pipeline Steps

**1. Download** — Stream audio to Yandex S3, create MongoDB story record, extract technical metadata via ffprobe.

**2. Audio Enhance** — ffmpeg filter chain: FFT denoising, bandpass, dynamic normalization. Output: 16kHz mono WAV.

**3. Transcribe** — faster-whisper medium with Silero VAD on Mini PC. Returns transcript with word timestamps.

**4. Metadata Save** — Write duration, format, S3 keys to MongoDB. Status → `transcribed`.

**5. AI Enrichment** — Three parallel GPT-4o calls: summary + tags, keywords, image generation prompt.

**6. Embeddings** — Transcript chunked (512 tokens, 64 overlap), batch-vectorized via BGE-M3. Upserted into two Qdrant collections: `story_chunks` and `story_summaries`.

**7. Image Generation** — Prompt from step 5 sent to Google Imagen (Vertex AI). Cover uploaded to S3. Status → `ready`.

### Reliability

Each step: read prerequisites from MongoDB → execute → write result → complete job. On failure, BullMQ retries with exponential backoff. Steps calling the Mini PC check `/health` first — if it's down, the job throws and queues for retry.

---

## Observability & Admin Panel

Each processor records `startedAt`, `completedAt`, `durationMs`, and `error` per step in the MongoDB `pipeline` field. No external metrics store needed.

**Two new Angular pages** in the existing admin app:

- **Batch Overview** — progress bar (queued / processing / ready / failed), average step durations, queue depth per step. Polled every 10s.
- **Story List** — paginated table with a 7-cell step status strip per row (green / yellow / red). Expandable step timeline with error details. **Retry** button re-enqueues from the failed step only.

**NestJS endpoints:** `GET /admin/pipeline/stats`, `GET /admin/pipeline/stories`, `POST /admin/pipeline/retry/:storyId`.

**Telegram watchdog** (cron every 5 min): alerts on stalled stories (`processing` + `updatedAt` > 20 min) and failure spikes (>3 failed in 30 min).

Bull Board at `/admin/queues` for raw queue inspection alongside the Angular UI.

---

## Outcome

A 20-minute story processes in ~12–15 minutes end-to-end. Self-hosted ML cuts variable costs by ~80% vs. managed equivalents. Observability runs on top of existing MongoDB data — no separate monitoring stack. The pipeline handles batches of 100+ stories reliably with per-step retry and full visibility into what's stuck and why.
