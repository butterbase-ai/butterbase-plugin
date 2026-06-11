---
name: meetings
description: Use when building features that join, record, or transcribe Zoom/Meet/Teams/Webex calls — meeting bots, call notetakers, sales-call summarizers, interview transcribers. Covers ctx.ai.meetings.start/get/stop/list, webhook setup, and credit metering.
---

# Meeting Bots on Butterbase

Guide for building features that spawn meeting bots via `ctx.ai.meetings`. Covers the four SDK methods, webhook handling, idempotency, and cost-aware design.

---

## 1. When to Use

Reach for this skill when your feature involves any of:

- **Meeting notetaker** — bot joins a recurring standup or sales call and stores a transcript.
- **Sales call enricher** — join a customer call, wait for `bot.done`, then run an LLM over the transcript to populate a CRM record.
- **Interview pipeline** — record and transcribe candidate interviews, then score against a rubric.
- **Voice-of-customer transcripts** — capture support or discovery calls; feed chunks into a RAG collection for future search.

If you need only audio/video file upload without a live meeting join, use `manage_storage` instead.

---

## 2. The Four SDK Methods

| Method | What it does | One-line example |
|--------|-------------|-----------------|
| `bb.ai.meetings.start(opts)` | Spawns a bot and returns immediately with `status: "joining"` | `const { data: bot } = await bb.ai.meetings.start({ meetingUrl, transcript: true })` |
| `bb.ai.meetings.get(id)` | Fetches current status + artifact URLs | `const { data } = await bb.ai.meetings.get(bot.id)` |
| `bb.ai.meetings.stop(id)` | Removes the bot from the call early | `await bb.ai.meetings.stop(bot.id)` |
| `bb.ai.meetings.list(opts)` | Lists bots with optional `status`, `limit`, `cursor` filters | `await bb.ai.meetings.list({ status: 'done', limit: 50 })` |

All methods return `{ data, error }` — always check `error` before using `data`.

---

## 3. Typical Flow

1. **Configure webhook** — call `manage_ai` with `action: "configure_meetings_webhook"`, passing `forward_url` pointing at a deployed Butterbase function or your own endpoint.
2. **Start the bot** — app calls `bb.ai.meetings.start(...)`. Status begins as `joining`.
3. **Bot joins** — status progresses `joining → waiting_room → in_call → recording`. The `bot.in_call_recording` event fires.
4. **Call ends** — bot status moves to `ended`, then `done` once artifacts are processed.
5. **Webhook fires `bot.done`** — your handler receives the event. The bot's `recordingUrl` and `transcriptUrl` are now populated.
6. **Fetch artifacts** — download the recording or transcript using the URLs from `bb.ai.meetings.get(id)`.
7. **Write to substrate / database** — store the transcript text, link to the relevant entity (deal, candidate, customer), or insert into a RAG collection.
8. **Trigger downstream extract** — run an LLM summarization, action-item extraction, or scoring job.

---

## 4. Webhook Handler Skeleton

Deploy this as a Butterbase function with `trigger: { type: "http", config: { method: "POST", auth: "none" } }`.

```typescript
export async function handler(req: Request, ctx: any): Promise<Response> {
  const event = req.headers.get('x-bb-event');
  const body = await req.text();
  const payload = JSON.parse(body);

  const botId: string = payload.data?.bot_id ?? payload.data?.id;

  // Idempotency: skip if we already processed this bot + event combo.
  const claimed = await ctx.idempotency.claim(`meeting:${botId}:${event}`);
  if (!claimed) {
    return new Response('duplicate', { status: 200 });
  }

  switch (event) {
    case 'bot.done': {
      // Artifacts are ready — store a reference or kick off extraction.
      await ctx.db.query(
        `UPDATE meeting_jobs SET status = 'done', bot_id = $1 WHERE external_ref = $2`,
        [botId, payload.data?.metadata?.jobRef ?? botId]
      );
      break;
    }
    case 'transcript.done': {
      // Transcript artifact URL is now available. Kick off downstream processing.
      await ctx.db.query(
        `INSERT INTO transcript_queue (bot_id, created_at) VALUES ($1, now())`,
        [botId]
      );
      break;
    }
    case 'bot.fatal': {
      await ctx.db.query(
        `UPDATE meeting_jobs SET status = 'fatal' WHERE bot_id = $1`,
        [botId]
      );
      break;
    }
    default:
      // Other events (bot.in_call_recording, recording.done, transcript.failed) — handle as needed.
      break;
  }

  return new Response('ok', { status: 200 });
}
```

Key points:
- Use `ctx.idempotency.claim(key)` — the meetings service may retry for up to 24 hours.
- Verify `x-bb-key-id` matches the first 16 characters of your current webhook secret to detect stale post-rotation events.
- Return `200` even for events you don't handle — a non-2xx response triggers a retry.

---

## 5. Cost-Aware Design

Before dispatching a bot in a user-pays context, call `estimateCost` and surface the projected charge:

```typescript
const { data: estimate } = await bb.ai.meetings.estimateCost({
  durationMinutes: 60,
  transcript: true,
});
// estimate.usd — show this to the user or check against a quota
```

**Rates (v1):**

| Dimension | Rate |
|-----------|------|
| Recording (mp4 or audio_only) | **$0.50/hr + markup**, prorated per second |
| Transcription | **$0.15/hr + markup**, prorated per second |

Both charges are applied against the app's AI credit balance once the bot reaches `done`.

For user-pays apps, hold a projected credit reservation at bot-start time and display a live cost counter using the bot's current duration (polled from `bb.ai.meetings.get`).

---

## 6. Vendor-Neutral Note

The underlying meetings infrastructure may change providers without notice. Do not depend on undocumented fields in the event payload or on provider-specific bot behavior.

---

## 7. What This Skill Does NOT Cover

- **Real-time streaming transcripts** — word-by-word transcript streaming during an active call is on the roadmap; not available in v1.
- **Multi-region bot routing** — all bots are spawned from US East in v1; per-region dispatch is on the roadmap.
- **Browser-side recording** — `ctx.ai.meetings` must be called from a serverless function or server-side code, not directly from the browser. Use a Butterbase function as a backend proxy.
