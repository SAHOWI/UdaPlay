# CHANGE_LOG

## 2026-10-08 — Fix: Cross-query context loss in Udaplay_02_starter_project.ipynb

### Problem

The inline `Agent` class defined in cell `29de2a6c` of `Udaplay_02_starter_project.ipynb`
had no cross-query memory. Every call to `invoke()` started with an empty message list, so
context-dependent follow-up questions like "When was it released?" could not be answered
because the agent had no knowledge of the prior exchange.

Symptom in the saved notebook output:
- Query 1 ("Who published GTA: San Andreas?"): 5 messages
- Query 2 ("When was it released?"): **3 messages** — started fresh; agent asked for
  clarification because it had lost the GTA context.
- Queries 3 and 4 similarly showed no growing history.

### Root cause

Three gaps in the notebook's local `Agent` class (which shadowed the imported `lib.agents.Agent`):

1. **`AgentState` TypedDict** — missing `session_id` field.
2. **`_prepare_messages_step`** — always rebuilt `[SystemMessage, UserMessage]` from scratch,
   discarding whatever history was in `state["messages"]`.
3. **`invoke()`** — hard-coded `"messages": []` and never read from or wrote to
   `ShortTermMemory`.

The correctly-implemented version in `lib/agents.py` already used `ShortTermMemory` and
carried previous messages forward, but the notebook's local class shadowed it entirely.

### Changes

#### File: `Udaplay_02_starter_project.ipynb`

**Cell `29de2a6c` (Agent class definition):**

| # | What changed | Why |
|---|---|---|
| 1 | Added `from lib.memory import ShortTermMemory` import | Required for session memory |
| 2 | Added `session_id: Optional[str]` to `AgentState` TypedDict | Tracks which session a run belongs to |
| 3 | Added `self.memory = ShortTermMemory()` to `__init__` | Creates the in-memory session store |
| 4 | Rewrote `_prepare_messages_step` to check `state.get("messages", [])` first and only prepend `SystemMessage` when the list is empty | Preserves accumulated history across calls |
| 5 | Rewrote `invoke()` to accept `session_id`, load `previous_messages` from the last run stored in `self.memory`, pass them as `initial_state["messages"]`, and call `self.memory.add(run_object, session_id)` after each run | Enables persistent, growing conversation history |
| 6 | Added `reset_session()` helper method | Allows clearing session state between test runs |

**Cell `7feb6231` (query loop):**

| # | What changed | Why |
|---|---|---|
| 1 | Defined `SESSION_ID = "udaplay_session"` constant | Makes the session identifier explicit |
| 2 | Changed `UdaPlay_agent.invoke(query)` → `UdaPlay_agent.invoke(query, session_id=SESSION_ID)` | Routes all queries through the same session so history is shared |
| 3 | Added inline comments marking context-dependent queries | Documents the test intent |
| 4 | Changed `i=i+1` → `i += 1` (style) | Minor readability improvement |

### Expected result after fix

Message counts now grow monotonically across queries (each new turn appends to prior history):

| Query | Before fix | After fix (approximate) |
|---|---|---|
| 1 — Who published GTA: San Andreas? | 5 | 5 |
| 2 — When was it released? | 3 (reset!) | 8+ |
| 3 — What platform was Super Mario 64 on? | 5 (reset!) | 11+ |
| 4 — Which other games on this platform? | 17 | 18+ |

Query 2 now correctly answers with the GTA: San Andreas release year (2004) without asking
for clarification, because the agent retains the full history from Query 1.

### Reference

The pattern used matches the existing implementation in `lib/agents.py` (`ShortTermMemory`,
`session_id`, and the `_prepare_messages_step` guard) which was correct but was being
shadowed by the notebook's local class.
