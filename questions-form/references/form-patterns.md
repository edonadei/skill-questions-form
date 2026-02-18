# Form Patterns Reference

Advanced patterns and edge cases for the questions-form skill.

> **Key rule:** All intermediate interactions (button taps, free-text, partial submit)
> must `editMessage` in place and respond with `NO_REPLY`. Never send a new chat
> message mid-form except on Cancel or after a successful Submit.

---

## Timeout Handling

If more than 10 minutes pass between sending the form and receiving a `form:submit`
callback, the form may be stale.

On the next `form:` interaction:
- Process it normally if `form_message_id` is still known
- If unknown (e.g. session restarted): tell the user the form expired and re-send it

On the next unrelated message:
- Ask: `"You have an incomplete form. Would you like to continue or start over?"`
  with two buttons: `[Continue]` / `[Start over]`

---

## Form Cancellation

Always include Cancel alongside Submit in the last keyboard row:

```json
[
  { "text": "✓ Submit", "callback_data": "form:submit" },
  { "text": "✗ Cancel", "callback_data": "form:cancel" }
]
```

On `form:cancel`:
1. Edit the form message to `❌ Cancelled.` (no buttons)
2. Send one follow-up: `"Form cancelled. Let me know how you'd like to proceed."`

---

## Dependent Questions

When question B should only appear after question A is answered with a specific value:

1. Render the initial form without Q_B in the keyboard
2. When Q_A is answered and its value triggers Q_B:
   - Add Q_B's rows into the keyboard and update the message text
   - Edit the form message in-place with the expanded keyboard
3. Update `form_state` to include Q_B's key, initialized to `null`
4. Submit validation must now include Q_B

Example:
- Q1: "Platform?" → `[Web] [Mobile]`
- If user picks "Mobile": add Q1a: "iOS or Android?" → `[iOS] [Android] [Both]`
- Edit message to show the new question row inserted after Q1's row

---

## Large Option Sets (>6 Options per Question)

Split into rows of 2–3 buttons each within the same keyboard. Keep "Other ✏️" on
its own final row for that question.

Example with 7 options for a "Framework?" question:

```json
[
  [
    { "text": "React",   "callback_data": "form:fw:react" },
    { "text": "Vue",     "callback_data": "form:fw:vue" },
    { "text": "Angular", "callback_data": "form:fw:angular" }
  ],
  [
    { "text": "Svelte",  "callback_data": "form:fw:svelte" },
    { "text": "Next.js", "callback_data": "form:fw:next" },
    { "text": "Nuxt",    "callback_data": "form:fw:nuxt" }
  ],
  [
    { "text": "Remix",   "callback_data": "form:fw:remix" }
  ],
  [
    { "text": "Other ✏️", "callback_data": "form:fw:other" }
  ]
]
```

---

## Multi-Select Questions

For questions where the user can pick multiple options, use a toggle pattern:

- `callback_data` format: `form:<qid>:toggle:<value>`
- Maintain a `Set` for that question in `form_state`
- Each click adds or removes the value from the set
- Re-render the keyboard in-place after each toggle (selected items get `✅ `)
- Include a "Done ✓" button to close the multi-select

```json
[
  [
    { "text": "✅ React", "callback_data": "form:fw:toggle:react" },
    { "text": "Vue",      "callback_data": "form:fw:toggle:vue" }
  ],
  [
    { "text": "Angular",  "callback_data": "form:fw:toggle:angular" },
    { "text": "Svelte",   "callback_data": "form:fw:toggle:svelte" }
  ],
  [
    { "text": "✓ Done selecting", "callback_data": "form:fw:done" },
    { "text": "Other ✏️",         "callback_data": "form:fw:other" }
  ]
]
```

When the user taps "Done selecting":
- Record the current Set as the final answer for that question
- Collapse the multi-select rows to a single summary row showing selected items
- Edit the form message in-place → `NO_REPLY`

---

## Free-Text Disambiguation

When `awaiting_freetext` is set and the user sends a message:

- If the message is a `callback_data:` value (i.e. a button tap): process it
  normally as a button click. **Keep `awaiting_freetext` active** — the user still
  needs to type their answer.
- If the message is plain text: treat it as the free-text answer.
  - Record: `form_state[awaiting_freetext] = <user text>`
  - Clear `awaiting_freetext`
  - Edit the form message to show the typed value in the relevant row
  - `NO_REPLY`

---

## Resuming Interrupted Forms

If the session is reset or `form_message_id` is lost:
- The existing Telegram message still exists but the model has no reference to it
- On the next `form:` callback: respond with:
  `"It looks like I've lost track of the previous form. Let me re-send it."`
- Re-send the full form from scratch

---

## Partial Submission

If the user taps Submit with unanswered questions:
1. Edit the form message to append a warning line:
   `⚠️ Still needed: <question labels>`
2. Do **not** send a new message
3. `NO_REPLY`

The existing buttons remain active — user can answer remaining questions and tap
Submit again.

---

## Complete Example: 3-Question Form

### Questions

1. Project type: Web App / Mobile / API / Other
2. Timeline: This week / This month / No rush / Other
3. Budget: < $1k / $1k–5k / $5k–10k / > $10k / Other

### Initial message (1 message total)

```
text:
  📋 A few questions before we proceed — tap to answer, then Submit.

  1️⃣ What type of project?
  2️⃣ What is your timeline?
  3️⃣ Budget range?

buttons:
  [Web App]  [Mobile]  [API]
  [Other ✏️]
  [This week]  [This month]  [No rush]
  [Other ✏️]
  [< $1k]  [$1k–5k]
  [$5k–10k]  [> $10k]
  [Other ✏️]
  [✓ Submit]  [✗ Cancel]
```

### Interaction sequence (all via editMessage + NO_REPLY until submit)

1. User taps "Mobile" → `callback_data: form:type:mobile`
   → `form_state.type = "mobile"`
   → Edit message: Q1 line becomes `1️⃣ What type of project? → *Mobile* ✅`
   → "Mobile" button becomes `✅ Mobile`
   → `NO_REPLY`

2. User taps "Other ✏️" on Q2 → `callback_data: form:timeline:other`
   → `awaiting_freetext = "timeline"`
   → Edit message: Q2 line becomes `2️⃣ What is your timeline? → ✏️ type in chat ↓`
   → `NO_REPLY`

3. User types: `"End of March"`
   → `form_state.timeline = "End of March"`, `awaiting_freetext = null`
   → Edit message: Q2 line becomes `2️⃣ What is your timeline? → *End of March* ✅`
   → `NO_REPLY`

4. User taps "$1k–5k" → `callback_data: form:budget:1k_5k`
   → `form_state.budget = "1k_5k"`
   → Edit message: Q3 line becomes `3️⃣ Budget range? → *$1k–5k* ✅`
   → `NO_REPLY`

5. User taps "✓ Submit" → `callback_data: form:submit`
   → All answered: `{ type: "mobile", timeline: "End of March", budget: "1k_5k" }`
   → Edit message: replace with `✅ Form submitted. Processing your answers...`
   → Model proceeds with full reply using collected data
