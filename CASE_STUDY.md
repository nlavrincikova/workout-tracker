# Two Production Incidents — A Short Debugging Log

Two real incidents from running this system, kept short on purpose: what broke, how I found it, what I did.

## 1. A parallel write leaked raw data into the chat

Adding a new exercise via chat started replying with raw JSON —
`{"row_number":101,"exercise_id":100,"exercise_name_normalized":"lunge"}` — instead of the workout.

**Root cause:** the workflow replies with whatever node finishes last. Creating a new catalog exercise forked
into two parallel steps: write the new ID to the sheet, and resume building the reply. The sheet write was a
dead end with nothing after it — when it happened to finish last, its own output became the chat reply
instead of the workout. The exercise itself was created correctly; only the response was wrong.

**Fix:** made the two steps run in sequence instead of in parallel, so the reply-building step is always
last. No other logic changed. Verified on a duplicate of the workflow, then confirmed in production.

## 2. Chased the documented fix, and it didn't hold

The obvious "correct" fix for the incident above was to stop depending on node execution order at all — n8n
supports a response mode built for exactly that, and its own docs say it works with the setup I had.

I built it, tested it in the workflow editor, and it worked. Then I tested it through the actual application,
and it didn't: the page got an immediate placeholder response instead of the real one, and the workflow hung
waiting to reply. The editor test had passed for a reason that didn't hold outside the editor.

I reverted it the same day rather than debug a documented feature further, and left a clear note for why it's
off the table. Not every fix that matches the documentation is worth finishing.

## What this shows

Both incidents came from the same source: a fix depends on which piece finishes first, or which environment
you test in. Once I recognized that pattern, I found — and fixed — two more failures shaped the same way
elsewhere in the same codebase.

---

Full technical detail: [`DESIGN.md`](./DESIGN.md).
Data this agent produces, and what auditing it found: [workout-analysis](https://github.com/nlavrincikova/workout-analysis).
