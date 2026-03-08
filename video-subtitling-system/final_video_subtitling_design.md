# Video Subtitling Platform on GCP — Final Solution Design

## 1. Goal

Build a web application where a user uploads an MP4 movie, selects the target subtitle language, and later receives:

- a browser notification if they keep the page open
- an email with a link to the finished output

The system must prioritize **subtitle quality**, **large-file handling**, and **scalable parallel processing**.

---

## 2. High-level architecture

### User-facing path
1. User creates a job in the web app.
2. API creates a job record in Firestore and returns a **signed resumable upload URL** for Google Cloud Storage (GCS).
3. Browser uploads the MP4 **directly to GCS**.

### Processing path
1. When the browser upload completes, **GCS object finalize** emits the authoritative completion event.
2. A lightweight validator service checks the finalized object generation, size, checksum, object path, and Firestore job state.
3. If validation succeeds, the service transactionally updates Firestore to `UPLOADED` / `QUEUED_TRANSCRIPTION` and publishes a transcription message to **Pub/Sub**.
4. A **GPU transcription worker pool** on Compute Engine picks up the job.
5. The GPU worker:
   - detects source language
   - extracts audio
   - transcribes with **Whisper / WhisperX**
   - stores aligned transcript artifacts in GCS
6. A **CPU text-processing worker pool** picks up the next step.
7. The text worker:
   - reconstructs sentences
   - translates with **OpenAI API** if needed
   - applies subtitle segmentation
   - stores subtitle artifacts in GCS
8. A **CPU rendering worker pool** picks up the final step.
9. The rendering worker burns subtitles into the video with `ffmpeg` or produces sidecar subtitle files.
10. Final output is saved to GCS.
11. Firestore job status is updated to `READY` or `FAILED`.
12. User gets:
   - web notification via Firestore listener
   - email with a link back to the authenticated job page

---

## 3. Storage and job state

### Cloud Storage
Use separate GCS prefixes or buckets for:

- `uploads/` — original MP4
- `work/` — temporary extracted audio and intermediate files
- `subtitles/` — `.srt`, `.vtt`, intermediate transcript JSON
- `outputs/` — final subtitled MP4

### Firestore
Store one document per job, for example:

```json
{
  "jobId": "job_123",
  "ownerUid": "user_456",
  "status": "CREATED",
  "stage": "CREATED",
  "inputObject": "gs://bucket/uploads/u1/job_123/input.mp4",
  "inputGeneration": null,
  "inputSizeBytes": null,
  "inputChecksumCrc32c": null,
  "sourceLanguage": "auto",
  "detectedLanguage": null,
  "targetSubtitleLanguage": "en",
  "mode": "auto",
  "progress": 0,
  "outputObject": null,
  "errorMessage": null,
  "attemptId": null,
  "workerId": null,
  "leaseExpiresAt": null,
  "lastHeartbeatAt": null
}
```

Suggested statuses:

- `CREATED`
- `UPLOADING`
- `UPLOADED`
- `QUEUED_TRANSCRIPTION`
- `TRANSCRIBING`
- `QUEUED_TEXT_PROCESSING`
- `TEXT_PROCESSING`
- `TRANSLATING`
- `QUEUED_RENDERING`
- `RENDERING`
- `READY`
- `FAILED`

Required ownership/security fields:

- `ownerUid` to bind each job to the authenticated user
- `inputGeneration`, `inputSizeBytes`, `inputChecksumCrc32c` to make upload finalization idempotent
- `attemptId`, `workerId`, `leaseExpiresAt`, `lastHeartbeatAt` to support long-running worker leases

### Firestore transaction and lease model
Firestore is the source of truth for stage ownership.

Each worker must claim a stage using a transaction with compare-and-set semantics:

1. read the job document
2. verify the job is still in the expected prior stage
3. set:
   - `stage`
   - `attemptId`
   - `workerId`
   - `leaseExpiresAt`
   - `lastHeartbeatAt`
4. commit only if the stage has not changed

Example:

- validator moves `CREATED` / `UPLOADING` to `UPLOADED`
- transcription worker claims `QUEUED_TRANSCRIPTION` -> `TRANSCRIBING`
- text worker claims `QUEUED_TEXT_PROCESSING` -> `TEXT_PROCESSING`
- rendering worker claims `QUEUED_RENDERING` -> `RENDERING`

Workers must heartbeat by extending `leaseExpiresAt` and updating `lastHeartbeatAt`.
If a worker dies and the lease expires, a new worker may safely reclaim the stage with a new `attemptId`.
Final state updates must also verify that the `attemptId` still matches, so a stale worker cannot overwrite a newer attempt.

---

## 4. Browser notification mechanism

The frontend uses the **Firestore JavaScript SDK** with `onSnapshot()` on the job document.

Example shape:

```javascript
import { doc, onSnapshot } from "firebase/firestore";

const unsub = onSnapshot(doc(db, "jobs", jobId), (snap) => {
  const job = snap.data();

  if (job.status === "READY") {
    showToast("Your video is ready");
  }
});
```

Why this approach:
- simple for the frontend
- no custom WebSocket server required
- immediate UI updates when job state changes

### Multi-user SaaS access control
This platform is a multi-user SaaS product, even if usage is free.

Therefore:

- every job document must include `ownerUid`
- Firestore security rules must allow users to read only their own jobs
- browser clients must never receive direct access to private output objects by default
- GCS buckets and objects remain private to service accounts and backend workers

### Download-link policy
Do not store a long-lived signed URL in Firestore.

Recommended approach:

1. completion email links the user back to the authenticated job page in the app
2. when the user requests download, the backend verifies `ownerUid`
3. the backend generates a short-lived signed URL on demand
4. signed URLs should expire quickly and be regenerated as needed

---

## 5. Language selection and auto-detection

### User input
The user selects **target subtitle language**.

### Source language
The source language can be:
- explicitly provided by the user
- or set to `auto`

### Auto-detection design
If source language is `auto`, the GPU worker detects language before full transcription.

Recommended approach:
1. extract a short audio sample from the beginning
2. run a lightweight language-identification step
3. if confidence is low, sample a few more windows from different parts of the movie
4. choose the dominant language

This can be done with:
- Whisper language identification
- or an **OpenAI API call** on a short initial transcript/sample for language classification

### Mode selection
- if `detectedSourceLanguage == targetSubtitleLanguage` → **transcription only**
- otherwise → **transcription + translation**

---

## 6. Why use WhisperX

Use **WhisperX** instead of plain Whisper as the primary transcription engine.

### Why
WhisperX improves the transcription pipeline by adding:
- **word-level timestamps**
- better segment timing alignment
- better support for long-form subtitle timing workflows

This matters because subtitle quality depends not only on text accuracy, but also on **timing precision**.

### Practical benefit
Whisper alone gives variable-length speech segments.
WhisperX refines those timings so the system can:
- reconstruct sentences more accurately
- split subtitles at better boundaries
- align translated subtitles more cleanly back to the movie timeline

---

## 7. Very large file handling

Large MP4 files should never pass through the backend API.

### Upload
Use:
- signed URL
- resumable upload
- direct browser → GCS transfer

### Processing large audio
If the transcription model or runtime cannot reliably process the full audio in one pass, split the extracted audio into fragments.

### Fragmentation strategy
Split audio into chunks such as:
- 5–20 minutes each
- with **overlap** between adjacent chunks, e.g. 10–30 seconds

Example:

- chunk 1: `00:00–10:00`
- chunk 2: `09:45–19:45`
- chunk 3: `19:30–29:30`

### Why overlap is needed
Without overlap:
- words near the boundary can be cut
- sentence starts and ends can be lost
- subtitle timing becomes worse

With overlap:
- transcription quality around boundaries improves
- deduplication logic can later merge overlapping text safely

---

## 8. Why translation must receive context

Do **not** translate subtitle cues independently.

If you translate short raw segments one by one, the model may:
- mistranslate fragments
- lose pronoun/reference context
- produce unnatural phrase boundaries
- mistranslate a sentence that was split across chunk edges

### Correct design
1. transcribe audio into timestamped segments
2. reconstruct full sentences / short dialogue paragraphs
3. send grouped text to translation
4. keep a mapping from translated sentences back to original timestamps
5. apply subtitle segmentation afterward

### Why this improves quality
Translation models perform better when they receive:
- a complete sentence
- neighboring lines
- speaker/dialogue context
- stable terminology

This is especially important when the original transcript was produced from chunked audio, because chunk edges can cut linguistic units in half.

---

## 9. Translation with OpenAI API

Use the **OpenAI API** for translation.

### Why OpenAI API
Compared with translating subtitle fragments directly, an LLM-based translation stage is better at:
- preserving conversational context
- resolving pronouns and references
- handling incomplete fragments when nearby context is supplied
- producing more natural subtitle text

### Recommended translation unit
Translate **sentence groups**, not subtitle cues.

For example, each translation request can include:
- current sentence group
- previous sentence group
- next sentence group if available
- source language
- target language
- glossary / style constraints if needed

### Recommended output
Ask OpenAI for **structured JSON** such as:

```json
{
  "sourceLanguage": "pl",
  "targetLanguage": "en",
  "items": [
    {
      "sentenceId": "s101",
      "translation": "In that case, we still need to verify the data."
    }
  ]
}
```

### Why structured output matters
It makes it easier to:
- keep translation aligned to source sentence IDs
- rebuild subtitles deterministically
- avoid parsing fragile free-form text

---

## 10. Speech-aligned segments and subtitle segmentation algorithm

### Key principle
Use **speech-aligned timestamps** from WhisperX as the timing backbone.

### Why not fixed 5-second subtitles
Fixed windows create poor subtitle quality because they ignore:
- speech rhythm
- pauses
- sentence boundaries
- reading speed

### Better design
1. WhisperX generates word-level or improved segment-level timestamps
2. build sentence-level transcript units from those timings
3. if translation is needed, translate sentence-level units
4. run a **subtitle segmentation algorithm**
5. generate final `.srt` / `.vtt`

### What the subtitle segmentation algorithm should do
It should:
- avoid splitting text in the middle of a phrase
- respect reading speed limits
- keep subtitles within max characters per line
- keep 1–2 lines per cue
- enforce minimum and maximum cue duration
- use WhisperX word timings to choose natural break points

### Why WhisperX is important here
Because subtitle segmentation is strongest when it can rely on:
- word timestamps
- pause positions
- phrase boundaries inferred from speech timing

That produces much more natural subtitles than naive segment-to-SRT conversion.

---

## 11. Worker pool separation: GPU transcription, CPU text processing, and CPU rendering

Use **three separate worker pools**.

### A. GPU transcription pool
Runs on Compute Engine instances with GPUs.

Responsibilities:
- audio extraction
- language detection
- Whisper / WhisperX transcription
- writing aligned transcript artifacts to GCS

Why GPU pool should be separate:
- Whisper / WhisperX is GPU-intensive
- GPU machines are expensive
- transcription throughput should scale independently from text processing and rendering throughput

### B. CPU text-processing pool
Runs on standard CPU instances or container workers.

Responsibilities:
- sentence reconstruction
- translation request orchestration
- subtitle segmentation
- writing subtitle artifacts to GCS

Why this pool should be separate:
- translation and segmentation are not GPU-bound
- OpenAI API latency should not hold expensive GPU instances open
- the translation backlog can scale independently from ASR and rendering

### C. CPU rendering pool
Runs on standard CPU Compute Engine instances or container workers.

Responsibilities:
- `ffmpeg` subtitle burn-in
- optional multiple output variants
- packaging final MP4
- upload final outputs to GCS
- finalize metadata updates

Why separate CPU pool helps:
- video rendering is often CPU-bound or I/O-bound
- it avoids wasting GPU time on post-processing
- the number of rendering workers can scale differently from transcription and text-processing workers

---

## 12. Queue design

Use at least three Pub/Sub topics or logical queues:

- `transcription-jobs`
- `text-processing-jobs`
- `render-jobs`

### Upload-finalize trigger
The source of truth for upload completion is **GCS object finalize**, not a browser callback alone.

Flow:

1. API creates the job and reserves the destination object path
2. browser uploads directly to that exact object path using resumable upload
3. GCS emits an `object.finalize` event when the object version is durably committed
4. validator service checks:
   - object path matches the job
   - object generation has not already been processed
   - file size is within product limits
   - checksum matches if available
   - Firestore status is still compatible with upload completion
5. only after successful validation does the backend enqueue transcription

This makes upload completion authoritative and idempotent.

### Transcription queue message
Prefer object references, not signed URLs.

Example:

```json
{
  "jobId": "job_123",
  "bucket": "video-subtitles-app",
  "objectPath": "uploads/u1/job_123/input.mp4",
  "sourceLanguage": "auto",
  "targetSubtitleLanguage": "en"
}
```

Why not signed URLs in messages:
- signed URLs can expire
- internal workers should use service account permissions
- bucket/object references are stable and cleaner

### Render queue message
Example:

```json
{
  "jobId": "job_123",
  "videoObject": "uploads/u1/job_123/input.mp4",
  "subtitleObject": "subtitles/u1/job_123/final.srt",
  "renderMode": "burned_in"
}
```

### Text-processing queue message
Example:

```json
{
  "jobId": "job_123",
  "alignedTranscriptObject": "work/u1/job_123/aligned-transcript.json",
  "sourceLanguage": "pl",
  "targetSubtitleLanguage": "en"
}
```

---

## 13. How to scale multiple transcriptions simultaneously

The system scales because uploads, transcription, and rendering are **decoupled**.

### Scaling strategy
- each validated uploaded movie becomes a queue message
- multiple GPU workers consume transcription jobs in parallel
- multiple CPU text workers consume translation / segmentation jobs in parallel
- multiple CPU workers consume rendering jobs in parallel
- all pools scale independently

### Concurrency rules
On GPU workers:
- process one or a very small number of jobs per GPU
- concurrency depends on:
  - chosen Whisper model
  - GPU VRAM
  - average audio length
  - whether WhisperX alignment is enabled

On CPU text-processing workers:
- run several jobs per instance, limited mainly by CPU, memory, and outbound API rate limits

On CPU rendering workers:
- run several render jobs per instance if CPU and disk allow it

### Reliability
Use:
- idempotent job stages
- deterministic output paths
- retry-safe worker logic
- dead-letter handling for repeated failures

---

## 14. How to configure GPU workers on Compute Engine

Use a **Managed Instance Group (MIG)** with an instance template.

### Instance template should define
- machine type
- attached GPU type and count
- boot disk size
- service account
- network/subnet
- startup script or prebuilt VM image
- local SSD or adequate persistent disk for temp media processing

### Software installed on the GPU VM
The VM image or startup script should install:
- NVIDIA driver and CUDA stack as required
- Python runtime
- Whisper / WhisperX dependencies
- `ffmpeg`
- Pub/Sub client
- logging/metrics agent
- worker process supervisor

### Worker startup behavior
On boot, the worker should:
1. register health
2. pull jobs from Pub/Sub
3. process one job at a time or according to configured concurrency
4. write heartbeat / progress metrics
5. acknowledge queue messages only after durable state updates

### Autoscaling
Configure the MIG to scale using:
- queue backlog metric
- or a custom Cloud Monitoring metric representing pending transcription jobs

Recommended controls:
- `min_replicas`: keep a small warm pool if latency matters
- `max_replicas`: cap cost
- cooldown period: avoid oscillation

---

## 15. Recommended end-to-end processing pipeline

### If source language equals target subtitle language
1. upload MP4 to GCS
2. wait for GCS object finalize validation
3. queue transcription job
4. detect language if needed
5. transcribe with WhisperX
6. queue text-processing job
7. reconstruct sentences
8. run subtitle segmentation
9. generate subtitle file
10. queue render job
11. render final output
12. store output and notify user

### If translation is needed
1. upload MP4 to GCS
2. wait for GCS object finalize validation
3. queue transcription job
4. detect language
5. transcribe with WhisperX
6. queue text-processing job
7. reconstruct sentence-level transcript
8. translate sentence groups with OpenAI API, including nearby context
9. map translations back to timing structure
10. run subtitle segmentation using WhisperX timings
11. generate subtitle file
12. queue render job
13. render final output
14. store output and notify user

---

## 16. Final design rationale

This design is optimized for:
- **subtitle quality**
- **large-file robustness**
- **accurate timing**
- **context-aware translation**
- **independent scaling of expensive and cheap compute stages**

### Why this combination works well
- **WhisperX** improves timing precision over plain Whisper
- **chunked audio with overlap** handles long media safely
- **sentence reconstruction + context-aware OpenAI translation** improves translation accuracy
- **subtitle segmentation after translation** preserves readability
- **GCS object finalize validation** makes upload completion authoritative
- **Firestore job leases** prevent duplicate long-running work during Pub/Sub redelivery
- **separate GPU, text-processing, and rendering pools** reduce cost and improve throughput
- **Firestore listeners** provide simple real-time browser notifications
- **Cloud Storage + signed resumable upload** handles very large files efficiently

---

## 17. Recommended future enhancements

Possible future improvements:
- glossary / terminology memory per customer
- speaker diarization if needed
- subtitle editing UI before final render
- multiple subtitle tracks in one output
- confidence scoring and manual review for low-confidence jobs
- additional output formats such as WebVTT and soft subtitle tracks
