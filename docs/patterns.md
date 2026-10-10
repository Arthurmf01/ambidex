# Patterns

Short, sanitised excerpts from the private codebase. Names and comments are lightly generalised; the logic is as it runs.

## 1. Never let model output reach a button unchecked

The weekly clean-up assistant returns JSON proposing changes to the task list. Before any proposal is shown, it is checked against the tasks that are really open. Anything that names a task that does not exist, an action outside the allowed set, or a "sort" for a task that is already sorted is discarded.

```ts
export function assembleCleanup(raw: string, tasksMd: string, todayIso: string): CleanupView | null {
  let obj: unknown;
  try { obj = JSON.parse(extractJson(raw)); } catch { return null; }
  const list = (obj as { suggestions?: unknown }).suggestions;
  if (!Array.isArray(list)) return null;

  const open = new Map(openTasks(tasksMd).map((t) => [t.id, t]));
  const seen = new Set<string>();
  const suggestions: CleanupSuggestion[] = [];

  for (const s of list as { id?: unknown; action?: unknown; objective?: unknown; reason?: unknown }[]) {
    if (!s || typeof s.id !== 'string' || typeof s.reason !== 'string') continue;
    const task = open.get(s.id);
    if (!task || seen.has(s.id)) continue;            // unknown or repeated id
    const reason = sanitiseCopy(s.reason).slice(0, 160);
    if (!reason) continue;

    if (s.action === 'drop' || s.action === 'done') {
      suggestions.push({ taskId: s.id, action: s.action, reason });
    } else if (s.action === 'sort' && !task.objective
               && typeof s.objective === 'string' && OBJECTIVES.includes(s.objective)) {
      suggestions.push({ taskId: s.id, action: 'sort', objective: s.objective, reason });
    } else continue;

    seen.add(s.id);
    if (suggestions.length >= MAX_SUGGESTIONS) break;
  }
  return { generatedOn: todayIso, suggestions };
}
```

The same check runs again at render time against the live file, so a proposal about a task that has since been completed simply disappears.

## 2. Prompt for evidence, and allow silence

The clean-up prompt tells the model that proposing nothing is a valid answer, and forbids the lazy heuristic.

```
Propose changes ONLY where the evidence in the lists below is clear.
Never propose dropping a task merely because it is old or hard.
If unsure, propose nothing.
Each reason: one plain sentence stating the evidence, not a generic remark.
```

## 3. One table for a state machine

Every ticket move goes through one function that consults one table. An illegal transition raises instead of being quietly written.

```python
VALID_TRANSITIONS = {
    "open": {"triaged", "blocked", "cancelled"},
    "triaged": {"in-progress", "blocked", "cancelled"},
    "in-progress": {"awaiting-review", "done", "blocked"},
    "awaiting-review": {"done", "in-progress", "cancelled"},
    "blocked": {"open", "triaged", "in-progress", "cancelled"},
    "done": set(),
    "cancelled": set(),
}
```

## 4. Cache the expensive pass, and keep the old one on failure

The clean-up pass costs one model call. It runs at most once a week, and a failed run keeps serving the previous result.

```ts
export function loadOrRunCleanup(cachePath: string, todayIso: string, run: () => CleanupView | null): CleanupView | null {
  let cached: CleanupView | null = null;
  if (existsSync(cachePath)) {
    try { cached = JSON.parse(readFileSync(cachePath, 'utf8')) as CleanupView; } catch { cached = null; }
  }
  if (cached && Array.isArray(cached.suggestions)
      && daysBetween(cached.generatedOn, todayIso) < CLEANUP_EVERY_DAYS) return cached;
  const fresh = run();
  if (!fresh) return cached;
  writeFileSync(cachePath, JSON.stringify(fresh, null, 2));
  return fresh;
}
```

## 5. Staleness is relative to the promise made

A task is stale when it has been open longer than its own urgency flag allows, measured from the day that flag was last set. Unknown age means no verdict.

```ts
// Days a task may stay open under each urgency flag before it counts as stale.
export const STALE_AFTER_DAYS: Record<string, number> = { '🔴': 7, '🟠': 14, '🟡': 21, '🟢': 28, '🔵': 30 };

const since = ages[t.id]?.flag ?? null;
if (!since) continue; // age unknown: do not guess that it is stale
const ageDays = Math.max(0, daysBetween(since, todayIso));
const limit = (t.urgencyEmoji && STALE_AFTER_DAYS[t.urgencyEmoji]) || DEFAULT_STALE_DAYS;
if (ageDays <= limit) continue;
```

## 6. Tool allow-lists per agent

The executor that attempts tasks from the page is launched with an explicit allow-list and no shell.

```ts
export const EXECUTOR_ALLOWED_TOOLS = 'Read,Write,Edit,Grep,Glob,WebSearch,WebFetch';
// ... passed as: --allowedTools <list> --disallowedTools Bash
```

The mail triage agent is launched with an empty allow-list. It reads text and writes text. A separate API client performs the I/O.

## 7. A send that data cannot authorise

Sending an email is the one dangerous operation in the mail client. It takes a capability token that cannot be produced by parsing JSON or reading a field, only by one function, which is called only where a verified human action has just happened. Text inside an email, however cleverly written, cannot turn itself into an approval.

```ts
// A capability token proving a human approved a send. Nominal type: the private brand cannot
// be produced by parsing JSON or reading a ticket field, only by approveSend() below.
declare const _humanApprovalBrand: unique symbol;
export interface HumanApproval { readonly [_humanApprovalBrand]: true }

// The ONLY constructor of a HumanApproval. Called exactly where a verified human action is established.
export function approveSend(): HumanApproval {
  return { } as HumanApproval;
}

async sendDraft(draftId: string, approval: HumanApproval): Promise<any> {
  if (!approval) throw new Error('sendDraft: refused, no human approval'); // runtime backstop
  return this.call('/drafts/send', { method: 'POST', body: { id: draftId } });
}
```

## 8. A clickable preview that cannot touch real data

To test the page locally, the real request router runs in-process with an in-memory key-value store and a fake of the storage API. Every button works; writes land in memory only. Any other outbound request is refused.

```ts
globalThis.fetch = (async (input, init) => {
  const url = String(input instanceof Request ? input.url : input);
  const m = url.match(/api\.github\.com\/repos\/[^/]+\/[^/]+\/contents\/([^?]+)/);
  if (!m) return new Response('blocked by local preview', { status: 503 });
  // ... serve reads from memory, apply writes to memory
}) as typeof fetch;
```
