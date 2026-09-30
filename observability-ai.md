# Observability: AI-Assisted Exploration

## 1. Tracing

[Trace Notes](observability-ai-lms-learn-ops-api-notes.md)

## 2. Comparison: My Diagram vs Claude's Diagram

**Matched:** Both traces follow the same layers and order: UI dialog -> fetch
helper -> router -> view -> database -> serializer -> UI, followed by a
separate GET request that refreshes the note list. Both correctly show the
same three database reads before the save in create() (student, coach, note
type), then the INSERT.

**Differed:**
- Claude split the Serializer into its own participant and added an explicit
  human actor (Coach) at the top of the diagram. I had folded the serializer
  into a self-arrow on the view, and started with a generic Browser actor.
- Claude's diagram explicitly modeled the N+1 query problem in list() with a
  loop block: for each note returned, the serializer makes a separate query to
  resolve note_type, coach, and coach.user (needed for the author property).
  My manual diagram collapsed the whole list read into a single "SELECT notes
  for student" arrow and missed this per-note query cost entirely.
- Claude also showed that create() triggers one extra query during
  serialization (SELECT auth_user for author), a detail my diagram left out.
- Nothing in Claude's trace was missing or out of order compared to mine; it
  was consistently more detailed, especially around database query costs.