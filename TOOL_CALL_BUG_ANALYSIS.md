# Tool Call Loss Bug Analysis

**Date:** 2026-03-15
**Status:** Root cause identified, fix implemented

---

## Symptoms

1. Tool calls disappear when model sends JSON completion before function name arrives
2. Thinking tokens appear to be "consumed" by tool calls
3. Tool calls attempted inside regular messages cause messages to stop
4. Observed with: step-3.5-flash (immediate), qwen-oauth (after extended use)

---

## Root Cause

### The Bug Location
`packages/core/src/core/openaiContentGenerator/streamingToolCallParser.ts`

### The Bug Mechanism

When chunks arrive in this order:
1. Arguments arrive (with ID, no name)
2. JSON completes (depth=0, still no name)
3. Name arrives later (no ID, no arguments)

The parser incorrectly reassigns the name to a NEW index:

```
Chunk 1: addChunk(0, '{"file":', 'call_abc', undefined)
  → meta[0] = { id: 'call_abc' }

Chunk 2: addChunk(0, '"test.txt"}', undefined, undefined)
  → meta[0] = { id: 'call_abc' }
  → buffer[0] = '{"file":"test.txt"}'
  → depth[0] = 0 (JSON complete!)

Chunk 3: addChunk(0, '', undefined, 'write_file')
  → Parser sees index 0 has complete JSON (depth=0)
  → Calls findMostRecentIncompleteIndex()
  → Considers index 0 "complete" (BUG: doesn't check meta.name!)
  → Returns NEW index via findNextAvailableIndex()
  → meta[1] = { name: 'write_file' }  ← Name stored at WRONG index!
  → buffer[1] = ''                     ← Empty!

getCompletedToolCalls() returns []  ← Tool call LOST!
  - Index 0: has buffer, no name → skipped
  - Index 1: has name, no buffer → skipped
```

### The Flawed Logic

In `findMostRecentIncompleteIndex()`:

```typescript
// Check if this tool call is incomplete
if (meta?.id && (depth > 0 || !buffer.trim())) {
  maxIndex = Math.max(maxIndex, index);
}
```

This considers a tool call "complete" if:
- depth === 0 (JSON closed)
- buffer is non-empty
- meta.id exists

**It does NOT check for `meta.name`!** A tool call without a name is incomplete and cannot be emitted.

---

## Why Wasn't This Noticed Earlier?

The bug was introduced on **September 9, 2025** but not identified until **March 15, 2026** (6 months later).

### Observed Behavior

1. **step-3.5-flash**: Bug triggers immediately
2. **qwen3.5**: Bug triggers after extended sessions (context builds up)

### User's Observation

Bug correlates with context usage:
- step-3.5-flash has smaller context → hits pressure immediately
- qwen3.5 has larger context → hits pressure after extended use
- **Hypothesis:** Context pressure causes models to send name chunk after JSON completes

### Why It Was Missed

1. **Bug manifests as "model hangs"** - When tool call is lost, model waits for response that never comes. Looks like "model is thinking" not "tool call parsing bug"

2. **Related fixes masked the root cause:**
   - Sep 19, 2025: "fix: missing tool call chunks for openai logging"
   - Feb 12, 2026: "fix(openai): tool call cleanup order when fixing streaming errors"
   - These addressed symptoms, not root cause

### Timeline

| Date | Event |
|------|-------|
| 2025-09-01 | Refactor begins (fe270b55b) |
| 2025-09-09 | Refactor merged (#501) - **bug introduced** |
| 2025-09-19 | "fix: missing tool call chunks" - symptom addressed |
| 2026-02-12 | "fix: tool call cleanup order" - symptom addressed |
| 2026-02-28 | "fix: detect truncated tool call output" (a6cbb8e11) |
| 2026-03-15 | Root cause identified and fixed |

---

## Related Commits

| Commit | Date | Description |
|--------|------|-------------|
| `51b5947627` | 2025-09-09 | Refactor: emit only at finish_reason (bug introduced) |
| `36f6967a5` | 2025-09-08 | Fix: parallel tool calls with irregular chunks |
| `a6cbb8e11` | 2026-02-28 | Fix: detect truncated tool call output |
| `32dc43a38` | 2026-03-15 | Debug: remove emittedIndices (wrong fix) |

---

## Test Case

```javascript
const parser = new StreamingToolCallParser();

// Arguments arrive first (no name)
parser.addChunk(0, '{"file":', 'call_abc', undefined);
parser.addChunk(0, '"test.txt"}', undefined, undefined);

// Name arrives later (no arguments)
parser.addChunk(0, '', undefined, 'write_file');

// BUG: getCompletedToolCalls() returns []
// Expected: [{ id: 'call_abc', name: 'write_file', args: { file: 'test.txt' } }]
```

---

## Files Modified

| File | Change | Status |
|------|--------|--------|
| `converter.ts` | Emit during streaming + track emitted IDs | ✅ Fixed |
| `streamingToolCallParser.ts` | Consider `meta.name` in completeness checks | ✅ Fixed |
| `package.json` | Changed bin to qwen-local | 🔧 Local testing |

## Complete Fix

### 1. `streamingToolCallParser.ts` - Fix index reassignment bug

**Problem:** When name arrives after JSON completes, parser reassigns name to new index.

**Fix:** Consider a tool call "incomplete" if `meta.name` is missing:

```typescript
// findMostRecentIncompleteIndex():
if (meta?.id && (depth > 0 || !buffer.trim() || !meta.name)) {
  maxIndex = Math.max(maxIndex, index);  // Keep at same index
}

// findNextAvailableIndex():
if (!buffer.trim() || depth > 0 || !meta?.id || !meta.name) {
  return this.nextAvailableIndex;  // Index available for name to arrive
}
```

### 2. `converter.ts` - Emit during streaming + track IDs

**Problem:** Waiting until `finish_reason` allows parser state changes to lose tool calls.

**Fix:** Emit immediately when complete + has name, track IDs to prevent duplicates:

```typescript
// During streaming:
if (result.complete && result.value) {
  const meta = this.streamingToolCallParser.getToolCallMeta(index);
  if (meta.name) {
    parts.push({ functionCall: { ... } });
    if (meta.id) this.emittedToolCallIds.add(meta.id);
  }
}

// At finish_reason:
for (const toolCall of completedToolCalls) {
  if (toolCall.id && this.emittedToolCallIds.has(toolCall.id)) continue;
  // ... emit
}
```

---

## Verification

### Test case: name arrives after JSON completes

```javascript
const parser = new StreamingToolCallParser();

parser.addChunk(0, '{"file":', 'call_abc', undefined);      // args start
parser.addChunk(0, '"test.txt"}', undefined, undefined);    // JSON complete
parser.addChunk(0, '', undefined, 'write_file');            // name arrives

// BEFORE fix:
// - meta[0] = { id: 'call_abc' }
// - meta[1] = { name: 'write_file' }  ← WRONG INDEX!
// - getCompletedToolCalls() = []      ← LOST!

// AFTER fix:
// - meta[0] = { id: 'call_abc', name: 'write_file' }  ← CORRECT!
// - getCompletedToolCalls() = [{ id: 'call_abc', name: 'write_file', ... }]
```

### Test results

```
=== Chunk 3: name arrives ===
result: true
meta[0]: { id: 'call_abc', name: 'write_file' }  ← Stays at correct index!
getCompletedToolCalls(): [{ id: 'call_abc', name: 'write_file', args: {...} }]
```

---

## Lessons Learned

1. **"Complete" means "ready to emit"** - A tool call without a name is NOT complete, regardless of JSON state
2. **Index-based tracking is fragile** - The parser was reassigning indices based on incomplete criteria
3. **Defense in depth** - Both fixes are needed:
   - Parser fix: Keep name at correct index
   - Converter fix: Emit during streaming + track by ID
4. **Test with adversarial chunk ordering** - Most tests assume chunks arrive in logical order
5. **Streaming requires careful state management** - Emission timing matters when chunks arrive out of order

---

## Second Bug Discovered: Pipeline Finish Chunk Overwriting

### The Bug Location
`packages/core/src/core/openaiContentGenerator/pipeline.ts` - `handleChunkMerging()`

### The Bug Mechanism

When the API sends TWO finish_reason chunks:
1. First finish chunk arrives WITH tool calls → stored in `pendingFinishResponse`
2. Second finish chunk arrives WITHOUT tool calls → **OVERWRITES** the first one
3. Stage 2d yields the second (empty) finish response → tool calls lost

```
Chunk 1 (finish_reason=tool_calls, 3 tool calls):
  → handleChunkMerging stores in pendingFinishResponse
  → shouldYield=false (hold for merging)

Chunk 2 (finish_reason=tool_calls, 0 tool calls):
  → handleChunkMerging OVERWRITES pendingFinishResponse
  → shouldYield=false (hold for merging)

Stage 2d yields pendingFinishResponse:
  → Contains 0 tool calls (the second chunk)
  → Original 3 tool calls are LOST
```

### The Flawed Logic

In `handleChunkMerging()` (before fix):

```typescript
if (isFinishChunk) {
  // This is a finish reason chunk
  collectedGeminiResponses.push(response);  // ← OVERWRITES previous finish!
  setPendingFinish(response);
  return false;
}
```

The code unconditionally stored every finish chunk, even if a previous finish chunk already had tool calls.

### The Fix

Skip finish chunks that don't have tool calls when the pending finish already has them:

```typescript
if (isFinishChunk) {
  // Don't overwrite a pending finish that already has tool calls
  const lastCollected = collectedGeminiResponses[collectedGeminiResponses.length - 1];
  const pendingToolCallCount = lastCollected?.candidates?.[0]?.content?.parts
    ?.filter((p: Part) => 'functionCall' in p).length || 0;
  if (hasPendingFinish && pendingToolCallCount > 0 && !hasToolCalls) {
    return false;  // Skip this finish chunk
  }
  collectedGeminiResponses.push(response);
  setPendingFinish(response);
  return false;
}
```

### Observed Pattern in Logs

```
[CONVERTER]   getCompletedToolCalls returned 3 tool calls
[CONVERTER]   Emitting tool call: id=call_xxx, name=glob
[CONVERTER]   Emitting tool call: id=call_yyy, name=glob
[CONVERTER]   Emitting tool call: id=call_zzz, name=glob
[PIPELINE] Stage 2d: YIELDING final pending finish response with 0 parts (0 tool calls)
```

The converter emitted 3 tool calls into the first finish chunk, but the pipeline yielded a finish response with 0 tool calls because the second finish chunk overwrote the first.

### Why This Bug Was Missed

1. **Requires specific provider behavior** - Only providers that send finish_reason twice trigger this
2. **Symptom looks like other bugs** - Tool calls disappearing could be parser bug, converter bug, or pipeline bug
3. **Logging revealed the issue** - Only with detailed pipeline logging did we see the second finish chunk overwriting the first

---

## Complete Solution: Three Complementary Fixes

The two bugs are **complementary** - they address different failure modes:

| Failure Mode | Fix 1 (Parser) | Fix 2 (Converter) | Fix 3 (Pipeline) |
|--------------|----------------|-------------------|------------------|
| Name arrives AFTER JSON completes → parser reassigns index | ✅ Fixes | ✅ Mitigates | ❌ No effect |
| Two finish chunks → second overwrites first | ❌ No effect | ❌ No effect | ✅ Fixes |

### Fix Summary

| File | Change | Purpose |
|------|--------|---------|
| `streamingToolCallParser.ts` | Check `meta.name` in completeness checks | Prevents index reassignment when name arrives late |
| `converter.ts` | Emit during streaming + track emitted IDs | Emits tool calls immediately, prevents duplicates |
| `pipeline.ts` | Skip finish chunks without tool calls | Prevents second finish chunk from overwriting first |

All three fixes work together for comprehensive tool call reliability.
