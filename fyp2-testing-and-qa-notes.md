# FYP2 Testing and Quality Assurance Notes

## Purpose

This file captures the testing and quality assurance work completed during FYP2 for HudaAI. It is intended as source material for the final FYP2 report, especially Chapter 4 testing details and the evaluation/validation discussion.

The notes combine three evidence streams:

- Code-level testing discovered in the admin panel and transcription pipeline repositories.
- Manual and internal validation performed by the project team and the Alhuda review team.
- User acceptance testing reported by the project team, including beta testing with more than 3000 testers across Apple TestFlight and Google Play testing channels.

## Systems Reviewed

### Admin panel

Repository: `/Users/hashirayaz/Documents/projects/alhuda/qfa-admin-panel-nextjs`

The admin panel is a Next.js 16 application used by admins, reviewers, and experts to inspect generated content, review transcriptions, manage flags, trigger extraction jobs, and view basic operational metrics. Its tests are implemented with Vitest and Testing Library.

Relevant files and areas:

- `src/app/transcriptions/[id]/page.tsx`: transcript review, edit, flag, and fix workflow.
- `src/components/SegmentListItem.tsx`: segment-level editing and flag display.
- `src/components/FlagNavigation.tsx`: navigation and batch fixing for flagged segments.
- `src/components/TranscriptMetricsBanner.tsx`: per-transcript metrics and flag statistics.
- `src/services/segmentFlagService.ts`: flag category, flag creation, resolution, deletion, merge, and stats retrieval.
- `src/services/transcriptService.ts`: transcript save flow, versioning, flag continuity, dashboard stats, and short-content stats.
- `src/services/shortContentService.ts`: frontend job submission and polling for short-form content generation.
- `src/app/extract-short-content/page.tsx`: short-form extraction job page.
- `src/app/extract-short-content/[id]/page.tsx`: clip viewer and short-content metrics display.
- `src/app/api/short-content/[id]/route.ts`: frontend API route for fetching saved short-content metadata.

### Transcription pipeline

Repository: `/Users/hashirayaz/Documents/projects/alhuda/qfa-transcription-pipeline`

The transcription pipeline is an event-driven FastAPI and worker system. It handles audio transcription, LLM correction, entity extraction, heading extraction, short-form content generation, RAG ingestion, and RAG querying.

Relevant files and areas:

- `app/api/routers/short_content.py`: short-content queue and retrieval endpoints.
- `app/services/short_content_service.py`: short-form clip generation orchestration.
- `app/workers/short_content_worker.py`: RabbitMQ worker for clip extraction jobs.
- `tests/unit/test_short_content_logic.py`: unit tests for short-content helper logic.
- `tests/api/test_queue_endpoints.py`: API queueing tests, including short-content prerequisites.
- `tests/workers/test_worker_callbacks.py`: worker callback behavior and queue handling.
- `docs/testing.md`: testing guide describing the intended pytest structure.

## Automated Testing Strategy

The automated testing strategy covers both the frontend admin panel and backend pipeline. The two projects use different test runners because they are built on different stacks.

### Admin panel automated tests

The admin panel uses:

- Vitest as the test runner.
- Testing Library for React component behavior.
- Mocked Supabase and backend calls for API and service behavior.
- JSDOM for browser-like runtime behavior.

The test inventory includes API route tests, service tests, library tests, and component tests. The current `src` test files include:

- `src/app/api/ai/fix-segment/__tests__/route.test.ts`
- `src/app/api/ai/segments/batch/__tests__/route.test.ts`
- `src/app/api/ai/translate-segment-arabic/__tests__/route.test.ts`
- `src/app/api/transcription/__tests__/route.test.ts`
- `src/app/api/transcription/[id]/__tests__/route.test.ts`
- `src/components/SegmentListItem.test.tsx`
- `src/lib/__tests__/aiRouteSecurity.test.ts`
- `src/lib/__tests__/segmentAiActions.test.ts`
- `src/lib/__tests__/validateSegments.test.ts`
- `src/services/__tests__/downloadFormatService.test.ts`
- `src/services/__tests__/entityExtractionService.test.ts`
- `src/services/__tests__/transcriptionJobService.test.ts`

The frontend tests cover:

- API error handling for transcription and AI routes.
- OpenRouter timeout and fallback-key behavior for AI segment correction and translation.
- Batch AI segment handling where one failed item does not fail the whole batch.
- AI route security checks such as authentication and rate limiting.
- Segment validation for uploaded JSON transcripts.
- Download format generation.
- Entity extraction service behavior.
- Transcription job persistence and status updates.
- Segment list selection behavior used by the review and fix workflow.

### Pipeline automated tests

The pipeline uses pytest and organizes tests by marker:

- `unit`: pure logic with no external I/O.
- `api`: FastAPI endpoint tests with mocked dependencies.
- `service`: service orchestration tests with mocked Supabase, Redis, LLM, and SQLite dependencies.
- `worker`: RabbitMQ callback behavior.
- `integration`: API contract and compatibility checks.

The testing guide in `docs/testing.md` states that the suite is designed to run without Docker, real API keys, or external services, because the external systems are mocked. This is important for repeatability: tests can validate queue messages, status transitions, prompt structures, and response contracts without requiring live Supabase, Redis, RabbitMQ, OpenRouter, or Whisper infrastructure.

The current pipeline test files include:

- `tests/unit/test_short_content_logic.py`
- `tests/unit/test_merge_chunks.py`
- `tests/unit/test_build_prompt.py`
- `tests/unit/test_entity_keys.py`
- `tests/unit/test_quranlite_utils.py`
- `tests/unit/test_qfa_db_utils.py`
- `tests/unit/test_constants.py`
- `tests/unit/test_connection_pool.py`
- `tests/unit/test_regression.py`
- `tests/api/test_transcribe_endpoints.py`
- `tests/api/test_rag_endpoints.py`
- `tests/api/test_job_status_endpoints.py`
- `tests/api/test_queue_endpoints.py`
- `tests/services/test_rag_ingestion_service.py`
- `tests/services/test_rag_retrieval_service.py`
- `tests/services/test_entity_extraction_service.py`
- `tests/services/test_llm_correction_service.py`
- `tests/services/test_transcription_service.py`
- `tests/workers/test_worker_callbacks.py`
- `tests/integration/test_api_contract.py`

## Testing Coverage by Feature

### Short-form content generation

Short-form content generation is tested at multiple levels in the pipeline.

Unit tests in `tests/unit/test_short_content_logic.py` verify:

- Transcript segments are assigned to heading groups using segment midpoint.
- Segments outside defined heading ranges fall back to the last heading.
- Empty heading groups are removed before LLM processing.
- Heading groups are chunked without splitting a heading across chunks.
- Large heading groups remain intact even if they exceed the nominal chunk size.
- Clip candidates below the independence score threshold are removed.
- Clip candidates shorter than 60 seconds or longer than 720 seconds are removed.
- Overlapping clips are de-duplicated by keeping the higher-scored candidate.
- The `max_clips` limit is applied.
- The LLM input format contains heading titles, timestamps, segment blocks, and transcript text.

API tests in `tests/api/test_queue_endpoints.py` verify:

- `POST /extract-short-content` queues a message to the `short_content_jobs` RabbitMQ queue.
- The endpoint validates prerequisites before queueing.
- If headings or corrected transcript versions are missing, the endpoint returns a validation failure and does not publish a queue message.

Worker tests in `tests/workers/test_worker_callbacks.py` verify that the short-content worker follows the same reliability contract as other pipeline workers:

- Valid JSON message calls the service and acknowledges the RabbitMQ message.
- Invalid JSON is rejected without requeue.
- Missing required fields are rejected without requeue.
- Service failures are retried through the configured retry mechanism.

Service-level coverage is implied by the short-content unit tests plus API/worker tests. The end-to-end service itself includes several runtime safeguards:

- Redis job status updates for `pending`, `processing`, `completed`, and `failed`.
- Chunk progress updates such as `1/2` and `2/2`.
- LLM retry with exponential backoff and jitter.
- ffmpeg failure detection through non-zero exit code handling.
- Empty-result persistence when no clips pass threshold.
- Cleanup of temporary audio and clip directories in a `finally` block.

### Flagging and expert review

The FYP2 admin panel introduced a reviewer-oriented flagging workflow. The automated tests related to this area are mainly frontend and service tests.

Relevant tested areas:

- `SegmentListItem.test.tsx` validates segment row interaction in selection mode, including labeled checkboxes, toggle callbacks, disabled behavior, and memoization.
- `validateSegments.test.ts` validates JSON segment imports before they enter the review workflow.
- API route and AI route tests validate the failure handling for segment AI fixes and translations.
- `segmentAiActions.test.ts` and AI route tests support the fix pass because flagged categories may call AI-assisted repair logic.

Implementation-level checks in the code include:

- A flag cannot be created unless a user is authenticated.
- Flags are stored with transcript id, version number, version id, segment index, segment id, category id, optional note, creator, and resolution state.
- Resolving a flag records `resolved`, `resolved_by`, and `resolved_at`.
- Deleting a flag logs the user action when a user is present.
- Category merge validates source and target IDs, prevents self-merge, checks both categories exist and are active, moves existing flags, and deactivates the source category.

### Metrics and operational dashboard

The FYP2 frontend includes basic metrics for:

- Total transcriptions.
- Pending, approved, and rejected transcript review status.
- Total generated short clips.
- Number of transcriptions with generated short clips.
- Per-transcript segment count.
- Current transcript version and total versions.
- Transcript approval status.
- Total flags, unresolved flags, resolved flags, flagged segments, percent flagged, and category breakdown.

The dashboard and transcript metrics rely on service-layer Supabase queries and an RPC call:

- `getTranscriptionStats()` counts raw transcriptions and approval status.
- `getShortContentStats()` counts generated clips from `extracted_short_content`.
- `getTranscriptFlagStats()` calls `get_transcript_flag_stats` with transcript id and version number.

These metrics support quality assurance because the team can monitor how much generated content exists, how much is pending review, and which transcripts still have unresolved issues.

## Manual Testing and User Acceptance Testing

### Internal team testing

The team performed self-testing during implementation, especially around the review workflow and the short-content pipeline. This included:

- Triggering transcription and downstream extraction jobs from the admin panel.
- Checking job submission and job polling behavior.
- Reviewing generated transcript segments against audio playback.
- Flagging incorrect, unclear, mistranslated, or structurally problematic segments.
- Resolving flags after edits.
- Confirming that unresolved flags remain visible after saving a new transcript version.
- Checking short-content clip generation and playback in the admin panel.
- Inspecting clip titles, timestamps, durations, independence scores, and generated audio URLs.

### Alhuda team testing

The Alhuda team reviewed the system through their domain expertise. This testing is important because automated tests can validate mechanics, but only qualified reviewers can validate religious content quality, Urdu/Arabic correctness, and whether the selected short clips are meaningful and contextually safe.

The internal Alhuda review process focused on:

- Correctness of generated transcript text.
- Usability of the flagging interface.
- Clarity of issue categories.
- Whether experts can efficiently move between flagged segments.
- Whether AI-fixable categories produce useful suggested fixes.
- Whether some categories require manual review rather than automated fixes.
- Whether generated short clips are standalone enough to be shared with users.

### Beta user acceptance testing

The project team reports user acceptance testing with more than 3000 beta testers through Apple TestFlight and Google Play testing channels.

This beta testing validates the user-facing side of HudaAI at a scale that cannot be simulated through unit tests. The likely value of this testing includes:

- Confirming the lecture discovery and playback experience works for real users.
- Identifying issues in mobile navigation, search, playback, and transcript interaction.
- Observing whether users understand and benefit from generated lecture content.
- Testing deployment behavior across real iOS and Android devices.
- Validating that generated content can support broader audience use beyond internal reviewers.

For the final report, this should be framed as user acceptance testing rather than automated test coverage. It is evidence of usability and deployment readiness, while the code-level suites are evidence of implementation correctness and regression safety.

### Expert onboarding as quality assurance

FYP2 also added a process improvement: a team of experts was onboarded into the system for further quality assurance. This is a major validation step because the content domain is sensitive and requires expert review.

Expert onboarding supports QA in four ways:

- It distributes review work across more qualified users.
- It turns subjective review observations into structured flags.
- It creates a feedback loop where recurring flag categories reveal common model or pipeline weaknesses.
- It improves confidence before content is exposed to end users.

## Local Verification Notes

The following verification commands were attempted while preparing these notes:

### Pipeline

Command attempted:

```bash
python3 -m pytest --collect-only -q tests
```

Result:

```text
/opt/homebrew/opt/python@3.14/bin/python3.14: No module named pytest
```

Interpretation: the system Python in this environment does not have pytest installed. The repository contains a pytest suite and `pytest.ini`, but it could not be collected with the currently active Python environment.

### Admin panel

Command attempted:

```bash
npm test -- --run --reporter=dot
```

Result summary:

```text
Test Files  2 failed | 22 passed (24)
Tests       6 failed | 208 passed (214)
```

The failures came from files under `.worktrees/feat/merge-flag-categories/...`, including duplicate test paths and React hook/runtime conflicts caused by tests being collected from a nested worktree. This suggests the Vitest include/exclude configuration should exclude `.worktrees/**` during normal project test runs.

Important: these failures should not be described in the final report as feature failures. They are a local test discovery/configuration issue observed during this documentation pass.

## Report-Ready Summary

During FYP2, HudaAI moved from basic generated outputs toward a more mature quality assurance workflow. Automated tests cover backend queueing, worker behavior, short-content candidate filtering, API response contracts, AI route failures, JSON segment validation, and frontend review components. Manual testing by the project team and internal testing with the Alhuda team validated the religious and linguistic quality of generated content. User acceptance testing with more than 3000 beta testers on Apple TestFlight and Google Play provided broader validation of the user-facing mobile experience. Finally, onboarding experts into the review system created a scalable human-in-the-loop process for continuous quality improvement.
