---
name: questions-form
description: >
  Present multiple clarifying questions as an interactive Telegram form using
  inline buttons. Use when the agent needs to ask the user 2 or more clarifying
  questions before proceeding with a task, and wants to present them all at once
  in a structured form layout with selectable options and an "Other" free-text
  escape hatch. Triggers when: gathering multi-faceted requirements, onboarding
  flows, preference collection, or any scenario requiring structured
  multi-question input from the user via Telegram.
---

# Questions Form

Present a **single, self-contained Telegram message** that acts as an interactive
form. The user taps buttons to toggle answers; the message updates in-place on each
tap. The model is only invoked meaningfully on `form:submit`.

## Core Design Principle

**Do NOT send a separate acknowledgement message per button tap.**
Instead, after recording the answer, **edit the form message in-place** to reflect
the new state (checkmarks, selections). Reply with `NO_REPLY` to suppress any
additional model output. This eliminates wasteful model roundtrips for every click.

The full flow:
1. Model sends **one form message** with all questions + buttons
2. User taps buttons → model edits that same message silently → `NO_REPLY`
3. User taps Submit → model wakes fully, reads collected answers, proceeds

---

## When to Use

- You need 2 or more clarifying questions answered before proceeding
- Questions have enumerable options (with optional free-text fallback)
- The channel is Telegram

Do **not** use this pattern for a single yes/no question — just send a plain
message with two buttons.

---

## Form Protocol

### Step 1: Define Questions Internally

```
questions = [
  { id: "type",     label: "What type of project?",  options: ["Web App", "Mobile", "API"] },
  { id: "timeline", label: "What is your timeline?",  options: ["This week", "This month", "No rush"] },
  { id: "budget",   label: "Budget range?",            options: ["< $1k", "$1k–5k", "$5k–10k", "> $10k"] },
]

form_state = { type: null, timeline: null, budget: null }
form_message_id = null   // filled after sending the form
awaiting_freetext = null
```

### Step 2: Render the Form as a Single Message

Build the entire form — header, all questions with their current selections, and
Submit/Cancel — into **one message with one button keyboard**.

Use `✅` to mark the currently selected option, plain label for unselected.

**Initial render example:**

```
📋 A few questions before we proceed — tap to answer, then Submit.

1️⃣ What type of project?
2️⃣ What is your timeline?
3️⃣ Budget range?
```

Buttons:

```json
{
  "action": "send",
  "channel": "telegram",
  "message": "📋 *A few questions before we proceed* — tap to answer, then Submit.\n\n1️⃣ What type of project?\n2️⃣ What is your timeline?\n3️⃣ Budget range?",
  "buttons": [
    [
      { "text": "Web App",  "callback_data": "form:type:web" },
      { "text": "Mobile",   "callback_data": "form:type:mobile" },
      { "text": "API",      "callback_data": "form:type:api" }
    ],
    [
      { "text": "Other ✏️", "callback_data": "form:type:other" }
    ],
    [
      { "text": "This week",  "callback_data": "form:timeline:this_week" },
      { "text": "This month", "callback_data": "form:timeline:this_month" },
      { "text": "No rush",    "callback_data": "form:timeline:no_rush" }
    ],
    [
      { "text": "Other ✏️", "callback_data": "form:timeline:other" }
    ],
    [
      { "text": "< $1k",    "callback_data": "form:budget:lt_1k" },
      { "text": "$1k–5k",   "callback_data": "form:budget:1k_5k" }
    ],
    [
      { "text": "$5k–10k",  "callback_data": "form:budget:5k_10k" },
      { "text": "> $10k",   "callback_data": "form:budget:gt_10k" }
    ],
    [
      { "text": "Other ✏️", "callback_data": "form:budget:other" }
    ],
    [
      { "text": "✓ Submit", "callback_data": "form:submit" },
      { "text": "✗ Cancel", "callback_data": "form:cancel" }
    ]
  ]
}
```

After sending, save the returned `message_id` as `form_message_id`.

### Step 3: Handle Button Callbacks

When a message arrives that is a `callback_data:` value starting with `form:`:

#### Regular option — `form:<qid>:<value>` (value ≠ `other`, `submit`, `cancel`)

1. Record: `form_state[qid] = value`
2. Re-render the form message with the new selection marked `✅`
3. **Edit the form message in-place** using `editMessage`:

```json
{
  "action": "edit",
  "channel": "telegram",
  "messageId": "<form_message_id>",
  "message": "<updated form text with selections>",
  "buttons": [ /* full updated keyboard */ ]
}
```

4. Respond with `NO_REPLY` — do not send any additional message.

#### "Other" option — `form:<qid>:other`

1. Set `awaiting_freetext = qid`
2. Edit the form message to show `✏️ Type your answer for Q<n> in the chat ↓`
3. Respond with `NO_REPLY`

#### Free-text answer (when `awaiting_freetext` is set)

Triggered when the next plain-text message arrives:

1. Record: `form_state[awaiting_freetext] = <user's text>`
2. Clear `awaiting_freetext`
3. Edit the form message to show the typed value in the relevant row
4. Respond with `NO_REPLY`

#### Submit — `form:submit`

1. Check for any `null` values in `form_state`
2. If **incomplete**: edit the form message to add a warning like `⚠️ Still needed: Timeline, Budget` — do **not** submit yet. Respond with `NO_REPLY`.
3. If **complete**: proceed with collected answers in your normal reply. Remove the form buttons by editing to a confirmation:

```json
{
  "action": "edit",
  "channel": "telegram",
  "messageId": "<form_message_id>",
  "message": "✅ Form submitted. Processing your answers..."
}
```

Then continue with the task using `form_state`.

#### Cancel — `form:cancel`

1. Discard `form_state` and `form_message_id`
2. Edit the form message to show `❌ Cancelled.` (no buttons)
3. Send a brief follow-up: `"Form cancelled. Let me know how you'd like to proceed."`

---

## In-Place Re-Render: How to Reflect Selections

When re-rendering the keyboard after a selection, mark the chosen option with `✅`
prepended to its text, and leave others plain. The callback_data stays the same
(re-clicking toggles, overwriting the previous selection).

Example after user picks "Mobile" for Q1:

```json
[
  [
    { "text": "Web App",      "callback_data": "form:type:web" },
    { "text": "✅ Mobile",   "callback_data": "form:type:mobile" },
    { "text": "API",          "callback_data": "form:type:api" }
  ],
  ...
]
```

Also update the message text to reflect current state. Example:

```
📋 *A few questions before we proceed* — tap to answer, then Submit.

1️⃣ What type of project? → *Mobile* ✅
2️⃣ What is your timeline?
3️⃣ Budget range?
```

---

## Button Layout Rules

- Maximum 3 buttons per row for clean rendering
- Keep button labels under 20 characters
- Use Title Case for option labels
- "Other" button: always `"Other ✏️"` on its own row
- Group all questions' buttons in a single keyboard — separated by visual rows
- Submit/Cancel always on the last row together
- Selected option: prepend `✅ ` to the button text
- Telegram callback_data limit: 64 bytes — keep `form:<qid>:<value>` short

---

## Token Efficiency Summary

| Event              | Model invocation? | Output           |
|--------------------|-------------------|------------------|
| Button tap         | Yes (lightweight) | editMessage only → NO_REPLY |
| Free-text answer   | Yes (lightweight) | editMessage only → NO_REPLY |
| Submit (complete)  | Yes (full)        | Proceed with task |
| Submit (incomplete)| Yes (lightweight) | editMessage warning → NO_REPLY |
| Cancel             | Yes (lightweight) | editMessage + one follow-up |

The model is invoked on each interaction but produces **zero chat output** until
Submit, keeping the UX tight and avoiding message spam.

---

## Post-Submit Response Streaming (Future: sendMessageDraft)

After `form:submit`, when the model generates its full response, OpenClaw currently
streams it using repeated `editMessageText` calls (`streamMode: "partial"`).

Telegram Bot API 9.3 added **`sendMessageDraft`** — a native streaming API that
sends partial text as it's generated, without the `sendMessage` + `editMessageText`
hack. This is cleaner and avoids hitting Telegram rate limits on rapid edits.

**When available** (requires bot to have forum topic mode enabled via
`channels.telegram.forumTopicsEnabled: true` in OpenClaw config):
- OpenClaw will automatically use `sendMessageDraft` for the post-submit response stream
- Each token chunk is sent via `sendMessageDraft` with a stable `draft_id`
- The draft materializes into a real message when the response is complete

**No action needed in the skill** — this is handled at the OpenClaw layer transparently.
Track support: https://github.com/openclaw/openclaw/issues/15714

---

## Edge Cases

See [references/form-patterns.md](references/form-patterns.md) for:
- Timeout and abandoned form handling
- Dependent questions (show Q2 only after Q1 is answered)
- Large option sets (>6 options per question)
- Multi-select toggle pattern
- Resuming interrupted forms (form_message_id lost)
