# Case Study: Audio Story Ingestion Pipeline

**Role:** Solution Architect · Tech Lead  
**Domain:** AI/ML, Audio Processing, Semantic Search  
**Stack:** NestJS · BullMQ · FastAPI · faster-whisper · BGE-M3 · OpenAI GPT-4o · Qdrant · MongoDB · Yandex S3 · Cloudflare Tunnel

---

## Background

MDS Collection is an audio platform for original spoken-word content — stories narrated by authors and uploaded through a web interface. With ~12,000 users and a growing catalog, the team needed to solve a discoverability problem: audio content is opaque by default. Without transcripts, tags, and semantic indexing, users can only search by title or author. That's a dead end at scale.

The task was to design and implement a backend pipeline that takes a raw audio file URL and produces a fully indexed, enriched story record: transcript, AI-generated summary and tags, vector embeddings for semantic search, and a generated cover image. The pipeline needed to be reliable, cost-controlled, and resumable — not a fragile chain of synchronous calls.

This was both an architecture problem and an engineering execution problem. As Tech Lead, I owned the full stack from system design through implementation.

---

## Discovery: Questions Before Architecture

**What does "done" mean for a single story?**  
A story is considered ready when: transcript is stored, summary and tags are attached, the content is searchable by semantic query, and a cover image is available. All four must complete — partial state should be recoverable, not retried from scratch.

**What is the ingestion trigger and throughput?**  
Stories are ingested on demand via an HTTP endpoint, triggered by author upload. Volume at launch is low (tens per day), but the architecture should not require rework if it reaches hundreds. The design should scale horizontally without structural changes.

**Where does the ML inference run?**  
The platform operates on a lean infrastructure budget. A dedicated Mini PC (Intel N95, 16GB RAM) was available for self-hosted inference. Running faster-whisper and BGE-M3 locally eliminates per-call API costs for the two most compute-intensive steps. The tradeoff is operational dependency on a physical machine — addressed via Cloudflare Tunnel, systemd services, and BullMQ's built-in retry.

**What language is the content in?**  
Primarily Russian. This eliminates most managed transcription APIs from consideration — they handle Russian adequately but struggle with proper nouns, informal speech, and narration cadence. Whisper medium gives significantly better results for this content type.

**What does semantic search need to look like?**  
Two access patterns: story-level discovery ("find stories about loneliness in the city") and passage-level retrieval for future RAG use cases. This means two separate vector collections — one per story summary, one per text chunk — rather than a single flat index.

**What happens when a step fails?**  
Every step must be independently retryable without side effects. If transcription fails after the audio has already been enhanced and uploaded to S3, re-running the transcription step should not re-download or re-process the audio. This requires persistent intermediate state.

---

## Back-of-Napkin Estimates

Assumptions: average story duration 20 minutes, 50 stories/day at peak.

| Component | Cost basis | Daily estimate |
|---|---|---|
| faster-whisper (local) | ~2× realtime on N95 → ~10 min/story | $0 (hardware sunk) |
| BGE-M3 embeddings (local) | batch inference, ~1s per story | $0 (hardware sunk) |
| OpenAI GPT-4o enrichment | ~3K tokens/story × $0.005/1K | ~$0.75/day |
| Image generation (Nano Banana) | ~$0.02/image | ~$1.00/day |
| Yandex S3 storage | raw + enhanced audio + covers | ~$0.50/day growth |
| **Total variable cost** | | **~$2.25/day (~$68/month)** |

The decision to self-host Whisper and BGE-M3 cuts what would otherwise be the dominant cost line. At 50 stories/day, OpenAI Whisper API would add ~$15/day. At this margin, the operational overhead of the Mini PC pays for itself within the first week.

---

## Tradeoffs Considered

**BullMQ FlowProducer vs. custom orchestration**  
A simple approach would be a chain of service calls inside a single async function. This is easy to write and hard to operate — one failure loses all progress, retries re-run everything, and there's no visibility into which step is stuck. BullMQ FlowProducer models the pipeline as a dependency tree: each step is an independent job that runs only when its prerequisite completes. The tradeoff is added complexity in setup and a Redis dependency. Given that transcript generation alone can take 10+ minutes, the ability to retry individual steps without starting over is non-negotiable.

**Persistent intermediate state in MongoDB vs. job data only**  
BullMQ jobs carry data, but job data is ephemeral — completed jobs are pruned. Each processor writes its output to a `pipeline` field in the MongoDB story document. This serves two purposes: idempotency (re-running a step checks if it already completed) and observability (the full processing history lives in one place). The tradeoff is an extra write per step, which is negligible compared to the processing time.

**Self-hosted Whisper on Mini PC vs. managed transcription**  
Managed APIs (OpenAI Whisper, AWS Transcribe) offer reliability and no infrastructure to manage. For English content at high volume, they're the right call. For Russian narration at low-to-medium volume, the quality gap is meaningful and the cost gap is decisive. The risk is hardware availability — mitigated by BullMQ retry with exponential backoff: if the Mini PC is offline, jobs queue and retry automatically when it comes back.

**Cloudflare Tunnel for ML access vs. VPN or direct exposure**  
The Mini PC sits on a home/office network without a static IP. Options: VPN (WireGuard), direct port forwarding, or Cloudflare Tunnel. VPN requires persistent client configuration on the VDS. Port forwarding exposes the machine directly. Cloudflare Tunnel provides stable HTTPS access with no open ports and automatic reconnection on IP change. It runs as a systemd service and is transparent to the application layer. The tradeoff is a dependency on Cloudflare's infrastructure, which is acceptable given their uptime track record.

**BGE-M3 vs. OpenAI embeddings**  
OpenAI's text-embedding-3 models are competitive and simple to integrate. BGE-M3 is multilingual by design, produces 1024-dimensional dense vectors, and runs locally at zero marginal cost. For a Russian-language platform with semantic search as a core feature, BGE-M3 is the stronger choice. The tradeoff is a local inference dependency and slightly higher integration effort.

**Two Qdrant collections vs. one flat index**  
A single collection storing all chunks is simpler. But story-level discovery ("find me something about urban isolation") and passage-level retrieval ("find the paragraph that mentions a specific event") require different granularities. A summary-level collection gives fast, story-scoped results. A chunks collection enables precise retrieval for RAG. Keeping them separate avoids mixing result types and simplifies query logic. The tradeoff is twice the upsert operations and slightly higher Qdrant storage.

**ffmpeg audio enhancement before transcription vs. direct transcription**  
Raw audio files from user uploads vary widely in quality — background noise, uneven levels, recording artifacts are common. Whisper handles clean audio well; noisy audio degrades accuracy significantly. Running an ffmpeg filter chain (FFT denoising, bandpass, dynamic normalization) and converting to 16kHz mono before transcription consistently improves output quality. The tradeoff is an extra processing step and temporary storage of the enhanced file. For a content platform where transcript quality drives search quality, it's the right investment.

---

## Selected Architecture

### Infrastructure Layout

The system is distributed across three nodes with distinct roles:

**VDS (application server)** runs the NestJS API, BullMQ workers, Redis, Qdrant, and MongoDB. It handles all orchestration, business logic, and data persistence.

**Mini PC (ML inference server)** runs a FastAPI service exposing two endpoints: transcription (faster-whisper) and embedding (BGE-M3). It's accessed by the VDS over a Cloudflare Tunnel HTTPS connection authenticated by a shared internal token.

**Cloud storage (Yandex Object Storage)** holds raw audio, enhanced audio, and generated cover images. It's S3-compatible, so the same AWS SDK client works with a swapped endpoint.

### Pipeline Flow

The pipeline is initiated by a single HTTP call with a file URL, author, and title. From there, everything runs asynchronously.

**1. Download**  
The audio file is streamed directly from the source URL to Yandex S3 in multipart chunks — no full buffering in memory. A story record is created in MongoDB with initial metadata and `status: queued`. ffprobe extracts technical metadata in parallel.

**2. Audio Enhancement**  
The raw file is processed through an ffmpeg filter chain: FFT noise reduction, bandpass filtering to isolate the vocal frequency range, and dynamic normalization. Output is WAV at 16kHz mono — the optimal format for Whisper. For particularly difficult recordings, a neural vocal separation step (demucs) is available as a manual fallback.

**3. Transcription**  
The enhanced audio key is sent to the FastAPI service on the Mini PC. faster-whisper medium runs with Silero VAD filtering, which automatically removes silence and non-speech segments. Word-level timestamps are captured. For files exceeding ~45 minutes, the system can route to AWS Transcribe as a fallback — useful if the Mini PC is under load or the story is an outlier in length.

**4. Metadata Save**  
Duration (from Whisper output), format, sample rate, and S3 keys for both raw and enhanced files are written to the story document. Story status advances to `transcribed`.

**5. AI Enrichment**  
Three GPT-4o calls run in parallel via `Promise.all`: summary + tags (structured JSON), keywords for search index augmentation, and a cover image generation prompt. Parallelism here cuts wall-clock time to roughly the latency of a single call. All results are written to MongoDB.

**6. Embeddings**  
The transcript is split into overlapping chunks to preserve context at boundaries. All chunks plus the summary are sent to the Mini PC in a single batch request. BGE-M3 returns 1024-dimensional dense vectors. The NestJS service upserts two Qdrant collections: `story_chunks` (one point per chunk) and `story_summaries` (one point per story). Chunk IDs are deterministic UUIDs derived from storyId and chunk index — enabling safe re-upsert on retry.

**7. Image Generation**  
The prompt generated in step 5 is sent to Nano Banana's image generation API. The resulting image is uploaded to Yandex S3, and the story document is updated with the cover URL and `status: ready`.

### Reliability Model

Each step follows the same contract: read prerequisites from MongoDB, execute, write result to MongoDB, complete the job. If a step throws, BullMQ retries with exponential backoff up to the configured attempt limit. Because state is in MongoDB rather than job data, a retry finds the story exactly where it left off.

The Mini PC's availability is treated as unreliable by design. Steps that depend on it (transcription, embeddings) check `/health` before attempting inference. A failed health check throws immediately, triggering BullMQ retry — jobs queue silently and drain when the service recovers.

---

## What Was Left Out of Scope (Intentionally)

- **Search API and frontend** — the pipeline populates Qdrant and MongoDB; the query layer and UI are a separate concern
- **Author-facing upload interface** — ingestion is triggered via internal HTTP; the upload flow is outside pipeline scope  
- **Batch reprocessing** — re-enriching the existing catalog with updated models is a future operational task, enabled by the idempotent step design
- **Observability stack** — Pino structured logging per step and BullMQ Bull Board cover operational visibility at current scale; a dedicated metrics stack (Prometheus/Grafana) is a future concern

---

## Outcome

The pipeline delivers a fully indexed story record — transcript, summary, tags, vectors, cover — from a raw audio URL with no manual intervention. Processing time for a 20-minute story is approximately 12–15 minutes end-to-end, dominated by transcription.

The self-hosted ML inference decision reduces ongoing variable costs by roughly 80% compared to managed API equivalents. The BullMQ FlowProducer model gives reliable, independently-recoverable steps with no custom orchestration logic. The two-collection Qdrant layout is already serving both story discovery and a future RAG interface without schema changes.

The architecture runs comfortably on lean infrastructure and is ready to scale horizontally at the application layer without touching the pipeline design.
