# FYP2 Short-Form Content Pipeline Notes

## Purpose

This file documents the short-form content functionality introduced in FYP2. It is intended as source material for the final FYP2 report, especially the implementation, algorithm design, and testing sections.

The short-form content feature converts long Islamic lecture transcriptions into curated, shareable audio clips. The pipeline identifies self-contained sections, generates bilingual metadata, slices the original lecture audio, uploads clips, and exposes the results through the admin panel and backend APIs.

## Problem Addressed

Alhuda lectures are often long-form audio sessions. A single lecture may be one to two hours long, which makes it difficult for users with limited time to benefit from the content. FYP2 addresses this by introducing automated short-form content generation.

The goal is not simply to cut random audio windows. The system must select segments that:

- Are meaningful as standalone clips.
- Have clear start and end boundaries.
- Preserve religious and educational context.
- Are short enough for quick listening.
- Include human-readable titles and descriptions.
- Can be reviewed and served through the existing HudaAI content system.

## System Placement

The short-form pipeline is part of the existing event-driven transcription backend:

```text
Corrected transcript + extracted headings + source audio
    -> FastAPI short-content endpoint
    -> RabbitMQ short_content_jobs queue
    -> short_content_worker
    -> short_content_service
    -> LLM candidate evaluation
    -> ffmpeg audio slicing
    -> Supabase Storage upload
    -> Supabase metadata update
    -> Admin panel viewing and playback
```

The pipeline follows the same architecture pattern as the transcription, LLM correction, entity extraction, heading extraction, and RAG ingestion workers.

## Relevant Backend Components

Repository: `/Users/hashirayaz/Documents/projects/alhuda/qfa-transcription-pipeline`

### API router

File: `app/api/routers/short_content.py`

Responsibilities:

- Accept new short-content extraction jobs.
- Validate that prerequisites are available before queueing.
- Store initial job state in Redis.
- Publish jobs to RabbitMQ.
- Return job status during polling.
- Retrieve generated clips by transcription id, audio code, series code, or featured ranking.

Important endpoints:

- `POST /extract-short-content`
- `GET /extract-short-content/{job_id}`
- `GET /short-content/{audio_code}`
- `GET /short-content/series/{series_code}`
- `GET /short-content/featured`
- `GET /short-content/transcription/{transcription_id}`

### Worker

File: `app/workers/short_content_worker.py`

Responsibilities:

- Declare and consume the durable `short_content_jobs` queue.
- Parse job messages containing `job_id`, `transcription_id`, and `max_clips`.
- Call `process_short_content_job()`.
- Acknowledge successful jobs.
- Reject invalid JSON or malformed messages without requeue.
- Retry service failures by republishing with `retry_count`.
- Discard failed messages after configured retry exhaustion.

### Service

File: `app/services/short_content_service.py`

Responsibilities:

- Validate and fetch all source data from Supabase.
- Map transcript segments to heading ranges.
- Chunk heading groups for LLM context limits.
- Ask Gemini 2.5 Flash, through OpenRouter, to evaluate clip candidates.
- Filter and rank candidates.
- Download source audio.
- Slice selected clips with ffmpeg.
- Upload clip files to Supabase Storage.
- Save final clip metadata to `raw_transcriptions.extracted_short_content`.
- Update Redis job status throughout processing.
- Clean temporary files.

## Relevant Frontend Components

Repository: `/Users/hashirayaz/Documents/projects/alhuda/qfa-admin-panel-nextjs`

### Service layer

File: `src/services/shortContentService.ts`

Responsibilities:

- Check backend health before submitting or polling jobs.
- Submit jobs to `NEXT_PUBLIC_BACKEND_URL/extract-short-content`.
- Poll `GET /extract-short-content/{jobId}`.
- Store active job ids in localStorage under `short_content_jobs`.
- Remove completed or dismissed jobs from localStorage.
- Fetch saved short-content metadata from the local Next.js API route.

### Job submission page

File: `src/app/extract-short-content/page.tsx`

Responsibilities:

- Show eligible and ineligible transcriptions.
- Require that a transcription is selected before submission.
- Allow the reviewer/admin to choose `maxClips` from 1 to 10.
- Check backend availability.
- Submit a short-content extraction job.
- Poll active jobs every 3 seconds.
- Show job status as pending, processing, completed, or failed.
- Display progress returned by the backend, such as chunk progress.
- Persist job ids in localStorage so jobs can be restored after a browser refresh.

The page also computes lightweight operational counts:

- Total short-content jobs shown locally.
- Active jobs.
- Completed jobs.

### Clip viewer page

File: `src/app/extract-short-content/[id]/page.tsx`

Responsibilities:

- Fetch generated short-content clips for a transcription.
- Sort clips by start time.
- Show total clip count.
- Show lecture duration.
- Show model used for generation.
- Show generation date.
- Render a timeline overview showing where clips occur in the lecture.
- Display each clip title, Urdu title field, description, timestamps, duration, independence score, and audio player.

### Next.js API route

File: `src/app/api/short-content/[id]/route.ts`

Responsibilities:

- Query `raw_transcriptions.extracted_short_content`.
- Return an empty clip response if no clips have been generated yet.
- Return stored short-content metadata when available.

## Prerequisites

The short-form content pipeline requires two prerequisites:

1. Extracted headings must already exist for the transcription.
2. The transcript must have a corrected version, meaning `version_number >= 1`.

The backend validates these prerequisites before queueing. If either is missing, the API returns a validation error and does not publish a RabbitMQ job.

This design prevents wasted worker time. It also improves user feedback because the frontend can show which transcriptions are ready and which are missing headings or correction.

## Data Used by the Pipeline

The service reads from `raw_transcriptions`:

- `extracted_headings`
- `audio_url`
- `audio_code`
- `version_number`

The service reads from `transcript_version`:

- `transcript_segments` for the latest corrected version.

The service writes back to `raw_transcriptions`:

- `extracted_short_content`

The service uploads audio files to Supabase Storage:

- Bucket: `audio-clips`
- Path pattern: `{audio_code}/clip_{index}_{start}_{end}.mp3`

## Stored Output Format

The final extraction is saved as a JSON object similar to:

```json
{
  "clips": [
    {
      "title_en": "Example Clip Title",
      "title_ur": "Urdu title field",
      "description_en": "One-sentence description of the clip.",
      "start": 120.5,
      "end": 420.0,
      "duration_seconds": 299.5,
      "independence_score": 0.91,
      "audio_url": "https://supabase-storage-url/audio-clips/example.mp3"
    }
  ],
  "generated_at": "2026-04-17T10:35:00",
  "model": "google/gemini-2.5-flash",
  "lecture_duration_seconds": 3600.0,
  "total_clips": 5
}
```

The fields support both backend serving and frontend review. The `independence_score` gives reviewers a quick quality indicator for whether a clip can stand alone without extra lecture context.

## Algorithmic Flow

### Step 1: Submit extraction request

The frontend submits:

```json
{
  "transcription_id": "uuid",
  "max_clips": 5
}
```

The backend:

- Validates headings and corrected transcript availability.
- Creates a UUID job id.
- Writes a Redis status object at `short_content_job:{job_id}`.
- Publishes a message to RabbitMQ.
- Returns immediately to the frontend.

### Step 2: Consume queue message

The short-content worker reads the message from `short_content_jobs`.

Expected message:

```json
{
  "job_id": "uuid",
  "transcription_id": "uuid",
  "max_clips": 5
}
```

The worker calls `process_short_content_job()` and acknowledges only after successful service processing.

### Step 3: Fetch source data

The service fetches:

- Headings from `raw_transcriptions.extracted_headings`.
- Latest corrected segments from `transcript_version.transcript_segments`.
- Audio URL for downloading the original lecture.
- Audio code for storage path organization.

Lecture duration is calculated from the maximum segment end time.

### Step 4: Map segments to headings

Each transcript segment is mapped to a heading using its midpoint:

```text
midpoint = (segment.start + segment.end) / 2
```

If the midpoint falls inside a heading range, the segment belongs to that heading group. Empty heading groups are removed.

This produces units like:

```text
Heading A
  segment 1
  segment 2
  segment 3
  char_count = sum(text lengths)
```

The advantage of heading-based grouping is that candidate clips align with semantically meaningful lecture sections instead of arbitrary fixed-duration windows.

### Step 5: Chunk for LLM context limits

Heading groups are packed into chunks of up to about 20000 characters. A heading group is never split across chunks.

This protects the LLM call from oversized prompts while preserving section integrity.

### Step 6: LLM candidate evaluation

The service calls Gemini 2.5 Flash through OpenRouter using LangChain structured output.

The model returns a `ShortContentEvaluation` object containing candidate clips:

- `title_en`
- `title_ur`
- `description_en`
- `independence_score`
- `adjusted_start`
- `adjusted_end`

The structured Pydantic model reduces parsing ambiguity and makes the output safer to process.

### Step 7: Filter and rank clips

All candidates are pooled and filtered:

- Minimum independence score: `0.7`
- Duration range: 60 to 720 seconds
- Sort by independence score descending
- Remove overlapping clips by keeping the higher-scored one
- Keep the top `max_clips`

This step is deterministic and testable. It protects the final output from weak or redundant candidates.

### Step 8: Download lecture audio

The service downloads the original lecture audio to a temporary file.

It also handles a known URL issue where a `404` may be resolved by removing `/Media/` from the URL path.

### Step 9: Slice audio clips

For each selected candidate, the service calls ffmpeg:

```bash
ffmpeg -y -i input.mp3 -ss START -to END -c copy output.mp3
```

The `-c copy` setting avoids re-encoding. This makes clipping faster and preserves the original audio quality.

### Step 10: Upload clips

Each generated MP3 clip is uploaded to Supabase Storage in the `audio-clips` bucket.

The service retrieves a public URL for each uploaded clip and attaches it to the clip metadata.

### Step 11: Save metadata

The final `ShortContentExtraction` object is saved to `raw_transcriptions.extracted_short_content`.

Redis status is updated with:

- `status = completed`
- `total_clips`
- `supabase_updated = true`

If an error occurs, Redis is updated with:

- `status = failed`
- `error`
- `supabase_updated = false`

### Step 12: Cleanup

Temporary audio and clip directories are removed in a `finally` block, even when processing fails.

## Reliability and Error Handling

The pipeline includes several reliability mechanisms:

- Prerequisite validation before queueing.
- RabbitMQ durable queue pattern.
- Worker retry mechanism using `retry_count`.
- Redis job status updates at every major phase.
- LLM retries with exponential backoff and jitter.
- ffmpeg error detection.
- Empty result handling when no candidate meets the threshold.
- Temporary file cleanup.
- Frontend backend-health checks before submission and polling.
- Frontend polling failure handling after repeated failures.
- localStorage job persistence for browser refresh recovery.

## Testing Coverage

Short-form content generation is covered by:

- Unit tests for segment-to-heading mapping.
- Unit tests for chunking.
- Unit tests for clip filtering, ranking, overlap removal, and max clip limits.
- API tests for `POST /extract-short-content`.
- API tests for prerequisite failure.
- Worker callback tests for valid messages, invalid JSON, missing fields, and failure retry behavior.

Key file: `tests/unit/test_short_content_logic.py`

Tested behaviors include:

- Midpoint assignment to headings.
- Fallback assignment to the last heading.
- Empty group removal.
- Chunk size behavior.
- Score threshold filtering.
- Duration filtering.
- Overlap removal.
- Clip count limiting.
- LLM input formatting.

## Metrics Exposed for Review

The frontend exposes short-content metrics at two levels.

On the dashboard:

- Total generated short clips.

On the clip viewer:

- Number of clips.
- Lecture duration.
- Model used.
- Generation date.
- Clip position on a timeline.
- Per-clip duration.
- Per-clip independence score.

These metrics help admins and reviewers understand both pipeline output volume and clip-level quality.

## Report-Ready Summary

In FYP2, HudaAI introduced an automated short-form content pipeline that transforms corrected long-form lecture transcripts into curated audio clips. The system uses extracted headings to identify semantically meaningful lecture sections, evaluates candidate clips with Gemini 2.5 Flash through structured LLM output, filters candidates by independence score, duration, and overlap, slices the original audio with ffmpeg, uploads clips to Supabase Storage, and stores metadata for frontend display. The feature is integrated into the admin panel with job submission, polling, local job persistence, clip playback, timeline visualization, and quality metrics. Automated tests validate the core algorithmic steps, API queueing, prerequisite handling, and worker behavior.
