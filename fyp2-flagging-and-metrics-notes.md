# FYP2 Flagging and Metrics Notes

## Purpose

This file documents the flagging workflow and basic metrics introduced in FYP2. It is intended as source material for the final FYP2 report, especially the sections on expert review, human-in-the-loop quality assurance, and admin-panel improvements.

## Feature Summary

FYP2 added a structured review workflow where experts and reviewers can flag transcript segments, categorize issues, navigate flagged segments, resolve issues, and track review progress through metrics.

The main goal is to turn expert review from an informal correction process into a trackable quality assurance workflow.

Before this feature, reviewers could edit transcript text, but the system had limited support for:

- Marking a specific segment as problematic.
- Categorizing the type of issue.
- Tracking unresolved issues.
- Returning to unresolved issues after saving.
- Measuring how much of a transcript required expert intervention.
- Seeing which issue types occur most frequently.

FYP2 addressed these gaps by adding segment flags, stable segment IDs, flag categories, flag continuity across versions, category-level navigation, and metrics banners.

## Relevant Files

Repository: `/Users/hashirayaz/Documents/projects/alhuda/qfa-admin-panel-nextjs`

Core implementation files:

- `src/app/transcriptions/[id]/page.tsx`
- `src/components/SegmentListItem.tsx`
- `src/components/FlagNavigation.tsx`
- `src/components/ReviewModeSwitch.tsx`
- `src/components/TranscriptMetricsBanner.tsx`
- `src/components/FixPassHeader.tsx`
- `src/services/segmentFlagService.ts`
- `src/services/transcriptService.ts`
- `src/lib/ensureSegmentIds.ts`
- `src/types/types.ts`
- `src/app/flags/page.tsx`
- `src/app/api/flag-categories/merge/route.ts`

Supporting documentation:

- `docs/versioning-flags-wer-improvements.md`
- `docs/segment-id-flag-continuity-changelog.md`
- `docs/superpowers/specs/2026-05-02-merge-flag-categories-design.md`

## Data Model

### Transcript segment identity

FYP2 added stable IDs to transcript segments.

Current segment shape:

```ts
export interface TranscriptSegment {
  id: string;
  start: number;
  end: number;
  text: string;
}
```

This change is essential because a segment must be tracked even if its position changes after insertions, deletions, or edits. Without a stable `id`, flags would only be attached to a numeric index. That breaks when a reviewer inserts or deletes a segment before the flagged one.

Utility:

```ts
src/lib/ensureSegmentIds.ts
```

This function ensures that segments arriving from legacy uploads or backend services always have an ID before being saved.

### Segment flags

Current flag shape:

```ts
export interface SegmentFlag {
  id: string;
  transcript_id: string;
  version_number: number;
  segment_index: number;
  version_id?: string;
  segment_id?: string;
  category_id: number;
  category?: FlagCategory;
  note?: string;
  created_by: string;
  resolved: boolean;
  resolved_by?: string;
  resolved_at?: string;
  created_at: string;
}
```

Important fields:

- `transcript_id`: identifies the transcript.
- `version_number`: keeps compatibility with version-based lookup.
- `version_id`: links to a concrete `transcript_version` row.
- `segment_index`: legacy/index-based location.
- `segment_id`: stable segment-level identity.
- `category_id`: issue category.
- `note`: optional reviewer explanation.
- `resolved`: whether the issue has been fixed.
- `resolved_by` and `resolved_at`: resolution audit details.

### Flag categories

Current category shape:

```ts
export interface FlagCategory {
  id: number;
  name: string;
  label: string;
  color: string;
  is_active: boolean;
  description?: string;
  ai_prompt?: string;
  can_fix_with_ai: boolean;
}
```

Categories allow the review team to distinguish issue types. Examples might include timing issues, transcription mistakes, missing Arabic text, unclear wording, or categories that can be fixed by AI.

Important FYP2 additions:

- Category descriptions.
- Category colors.
- `can_fix_with_ai` flag.
- Category-specific `ai_prompt`.
- Category merge workflow.

## Review Workflow

### Pass 1: Flagging

The transcript editor supports a flagging mode where reviewers inspect segments and mark issues without immediately editing everything.

Reviewer actions:

- Open a transcript.
- Review each segment while listening to synchronized audio.
- Add a flag to a problematic segment.
- Choose a flag category.
- Add an optional note.
- Continue reviewing the transcript.

The UI stores each flag against the segment's stable ID. This makes the workflow robust when segment order changes later.

### Pass 2: Fixing

After flagging, reviewers can switch to a fix-oriented workflow.

Fixing actions:

- Navigate only flagged segments.
- Filter flags by category.
- Select all flagged segments in a category.
- Apply AI-assisted fixing for categories marked `can_fix_with_ai`.
- Review AI suggestions.
- Accept or discard suggestions.
- Resolve flags when issues are fixed.
- Manually edit segments for categories that cannot be safely fixed by AI.

This two-pass approach helps experts separate problem discovery from correction. It also supports team workflows where one reviewer flags issues and another expert fixes them.

## Flag Creation and Resolution

Service file: `src/services/segmentFlagService.ts`

### Create flag

`createFlag()`:

- Requires an authenticated user.
- Looks up the current `version_id`.
- Inserts a row into `segment_flags`.
- Stores transcript id, version number, segment index, segment id, category id, note, and creator.
- Logs a `flag_created` audit action.

### Resolve flag

`resolveFlag()`:

- Requires an authenticated user.
- Updates `resolved` to true.
- Stores `resolved_by`.
- Stores `resolved_at`.
- Logs a `flag_resolved` audit action.

### Delete flag

`deleteFlag()`:

- Deletes the selected flag.
- Logs a `flag_deleted` audit action when a user is present.

## Flag Continuity Across Versions

One of the most important FYP2 improvements is unresolved flag continuity.

Problem before the improvement:

- Flags were tied to `version_number` and `segment_index`.
- Saving transcript edits creates a new version.
- After saving, unresolved flags from the previous version were no longer visible in the latest version.
- If a reviewer fixed 10 out of 20 flags and saved, the remaining 10 flags could disappear from the working view.

Implemented solution:

- Each segment now has a stable `id`.
- Each new flag stores `segment_id` and `version_id`.
- When saving a new transcript version, `updateTranscriptSegments()` carries unresolved flags forward from the old version to the new version.
- Only flags whose `segment_id` still exists in the new segment list are copied.
- Flags for deleted segments are naturally dropped because the referenced content no longer exists.

Save-flow logic:

```text
Reviewer saves version N
    -> system creates version N + 1
    -> system fetches unresolved flags from version N
    -> system checks which flagged segment IDs still exist
    -> system copies valid unresolved flags to version N + 1
    -> reviewer sees remaining unresolved flags in the latest version
```

This is important for real expert review because reviewing a long lecture can take multiple save cycles.

## Category Management

File: `src/app/flags/page.tsx`

Admins can manage flag categories from the Flag Categories page.

Supported actions:

- Create category.
- Edit category label, internal key, color, description, AI-fix setting, and AI prompt.
- Delete category.
- Merge categories.

### Category merge

File: `src/app/api/flag-categories/merge/route.ts`

The merge API:

- Validates request body.
- Requires source and target IDs.
- Prevents merging a category into itself.
- Verifies source category exists and is active.
- Verifies target category exists and is active.
- Updates all `segment_flags` from source category to target category.
- Deactivates the source category.

This prevents category sprawl. If reviewers create duplicate categories, admins can consolidate them without losing existing flag data.

## Metrics Added in FYP2

FYP2 added basic metrics at three levels: dashboard metrics, transcript-level metrics, and flag-category metrics.

### Dashboard metrics

File: `src/app/dashboard/page.tsx`

The dashboard displays:

- Pending review count.
- Approved count.
- Total files.
- Total short clips.

Data comes from:

- `getTranscriptionStats()` in `src/services/transcriptService.ts`
- `getShortContentStats()` in `src/services/transcriptService.ts`

These metrics help admins understand the overall state of generated and reviewed content.

### Transcript metrics

File: `src/components/TranscriptMetricsBanner.tsx`

The transcript detail page displays:

- Total segment count.
- Current version.
- Total versions.
- Approval status.
- Number of open flags.
- Number of resolved flags.
- Number of flagged segments.
- Percentage of flagged segments.
- Category breakdown.

The percentage flagged is calculated as:

```text
flagged_segments / total_segments * 100
```

This gives a quick quality signal. A transcript with a high flagged percentage may need deeper review, while a transcript with few or no unresolved flags may be closer to approval.

### Flag stats RPC

Service call:

```ts
getTranscriptFlagStats(transcriptId, versionNumber)
```

This calls the Supabase RPC:

```text
get_transcript_flag_stats
```

Returned shape:

```ts
{
  total_flags: number;
  unresolved_flags: number;
  resolved_flags: number;
  flagged_segments: number;
  category_breakdown: [
    { id: number, label: string, color: string, count: number }
  ];
}
```

These statistics support both reviewer workflow and final report evaluation because they quantify the amount and types of expert intervention required.

### Short-content metrics

Although short-content metrics are covered in the short-form pipeline notes, they also appear in the overall metrics system:

- Dashboard shows total generated short clips.
- Clip viewer shows clip count, lecture duration, generation model, generation date, clip duration, and independence score.

These metrics link FYP2 content generation with admin-panel monitoring.

## Human-in-the-Loop Quality Assurance

The flagging workflow is a human-in-the-loop QA mechanism.

The automated pipeline can generate transcripts and short clips, but domain experts must validate:

- Correctness of religious terminology.
- Accuracy of Quranic and Hadith references.
- Whether the Urdu text is understandable.
- Whether the audio and transcript timestamps align.
- Whether short clips preserve context.
- Whether generated text requires correction before public use.

Flags make expert feedback structured and measurable. Instead of leaving review comments outside the system, experts can annotate the exact segment, category, and issue note.

## Expert Onboarding

FYP2 onboarded a team of experts into the system for additional quality assurance. This improves the project in two ways:

- It increases review capacity.
- It gives the system domain-grounded feedback from qualified reviewers.

The flagging system makes expert work visible and trackable. Reviewers can identify issues, admins can monitor unresolved work, and recurring issue categories can guide future model and prompt improvements.

## Testing and Validation

The flagging and metrics functionality was validated through:

- Project-team self-testing of flag creation, resolution, filtering, navigation, and save behavior.
- Internal Alhuda testing by reviewers and experts.
- Code review of services and components.
- Automated tests around segment selection, AI route behavior, uploaded segment validation, and service/API failure handling.
- Manual verification that unresolved flags are copied forward after transcript saves.

Relevant automated tests:

- `src/components/SegmentListItem.test.tsx`
- `src/lib/__tests__/validateSegments.test.ts`
- `src/lib/__tests__/segmentAiActions.test.ts`
- `src/app/api/ai/fix-segment/__tests__/route.test.ts`
- `src/app/api/ai/segments/batch/__tests__/route.test.ts`
- `src/app/api/ai/translate-segment-arabic/__tests__/route.test.ts`

## Limitations and Future Improvements

The current metrics are useful but still basic. Future improvements could include:

- Word error rate from raw transcript to final expert-approved transcript.
- Per-version diff history.
- Average time to resolve flags.
- Reviewer-level workload metrics.
- Category frequency over time.
- Model-quality dashboards showing which issue types occur most after automatic generation.
- Automatic alerts for transcripts with unusually high flag density.
- Separate approval gates for short-form clips.

These future improvements are already aligned with the direction described in `docs/versioning-flags-wer-improvements.md`.

## Report-Ready Summary

In FYP2, HudaAI introduced a structured expert-review workflow through segment-level flagging and metrics. Reviewers can flag problematic transcript segments, assign issue categories, add notes, navigate unresolved issues, resolve flags, and use AI-assisted fixes for selected categories. Stable segment IDs and flag continuity ensure unresolved flags remain attached to the correct content even after transcript versions change. The admin panel now exposes dashboard metrics, transcript-level flag metrics, category breakdowns, and short-content counts. Together, these features convert quality assurance from an informal editing process into a measurable human-in-the-loop review system.
