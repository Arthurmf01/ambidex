# Design decisions

## Choices that held

| Decision | Why | What it cost |
|---|---|---|
| A Markdown file in Git as the single source of truth | Every tool I already use can read and edit it. History, undo and audit come free. Agents and humans share one format. | No queries. Concurrency has to be handled by hand (read before write, retry on conflict). |
| PWA plus one iOS Shortcut, no native app | Free, instant to ship, no developer account. | No share-sheet target and no push on iOS, hence the Shortcut and Telegram. |
| Transcribe on the server | iOS Safari cannot do live speech recognition. Server-side Whisper is free and far better on names and mixed-language speech. | One round trip before the text appears. |
| The capture path never triages | A wrong automatic guess corrupts a file I trust. A raw line in an Inbox cannot be wrong. | An Inbox that needs reviewing, which became its own feature. |
| Agents draft, never send | A wrong draft costs ten seconds. A wrong sent message costs a relationship. | One tap of friction on everything outward-facing. |
| Generate daily, read tasks live | AI narrative is expensive and needs local data; the task list is cheap and must be current. | Two code paths that must agree on how a task line is parsed. They share one pure module. |
| Headless agent on a flat subscription, not a metered API | Predictable cost, and the same agent harness I use interactively. | Depends on the laptop being awake. A slept laptop means a missed run. |
| Pure functions over strings for every file transform | Sorting, archiving, reopening and id generation are all `text in, text out`, so they are trivially testable without a network. | Some repetition in how lines are walked and counted. |
| Stable, human-speakable task ids | I can say "do A07" to an agent. Ids survive rewording. | New ids must be allocated against both the live file and the archive so none is reused. |

## Things I got wrong

**Marking work as started before anyone had started it.** Captures were stamped "in progress" at ingest. The agent only collects queued lines, so those captures were never processed. It looked healthy because nothing errored. Fix: a line is only moved to in progress when a worker is attached; otherwise it stays queued.

**Allocating ids from one source.** Ticket handles were derived from the task file alone. Once tickets were closed and their lines tidied away, the next handle collided with an existing ticket. Fix: take the maximum across the task file and the ticket files.

**Assuming capture was the hard part.** Capture got to a few seconds quickly, and the list filled up. The real failure was review: I was saving many things and losing them in the pile. The "needs a decision" block, the dropped state and the Done view came from that.

**Hiding completed tasks.** Completed tasks vanished from the page. Without a visible past there is no sense of progress and no way to undo a mistaken tick. Fix: an archive with dates, a Done view, and one-tap reopen.

**Believing an estimate over a measurement.** I expected that searching mail and calendar would recover contact details for most people in the follow-up list. It recovered four of forty-five. The feature that mattered was the plain one: flag entries with no way to reach the person, and let me archive them in one tap.

## Principles I now apply

1. **State describes facts.** A status is only set by the thing that makes it true.
2. **Validate every model answer against real data** before it can drive an action.
3. **Unknown is a value.** When the system cannot tell, it says so and does nothing.
4. **Reversible by default.** Nothing an agent or a button does should be unrecoverable.
5. **Smallest toolbox.** Give each agent only the tools its job needs.
6. **Free pre-checks before paid calls.** Look at the queue before waking a model.
7. **Fail soft.** A missing section is better than a missing page.
