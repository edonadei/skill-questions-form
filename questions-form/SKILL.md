---
name: questions-form
description: >
  Present multiple clarifying questions as an interactive Telegram Mini App form.
  Use when the agent needs to ask the user 2 or more clarifying questions before
  proceeding with a task. The form opens inside Telegram as a native panel with
  proper radio buttons, selected states, and "Other" free-text inputs. All state
  stays client-side until the user taps Submit — then the agent receives every
  answer at once as a single JSON message. Triggers when: gathering multi-faceted
  requirements, onboarding flows, preference collection, or any scenario requiring
  structured multi-question input from the user via Telegram.
---

# Questions Form

Present multiple clarifying questions as a Telegram Mini App form.
The form opens as a native panel inside Telegram — all answers stay client-side
until the user taps Submit. The agent receives everything at once.

## When to Use

- You need 2 or more clarifying questions answered before proceeding
- Questions have enumerable options (with optional free-text fallback)
- The channel is Telegram

Do **not** use this for a single yes/no question — just send one message with inline buttons for that.

## Mini App URL

The form is hosted at:
```
FORM_BASE_URL = https://<owner>.github.io/skill-questions-form/form.html
```

Replace `<owner>` with the GitHub username where this repo is deployed.

## Form Protocol

### Step 1: Build the Form Config

Create a JSON config object with all your questions:

```json
{
  "questions": [
    {
      "id": "type",
      "text": "What type of project is this?",
      "options": [
        { "label": "Web App", "value": "web" },
        { "label": "Mobile App", "value": "mobile" },
        { "label": "API / Backend", "value": "api" }
      ]
    },
    {
      "id": "timeline",
      "text": "What is your timeline?",
      "options": [
        { "label": "This week", "value": "this_week" },
        { "label": "This month", "value": "this_month" },
        { "label": "No rush", "value": "no_rush" }
      ]
    },
    {
      "id": "budget",
      "text": "What is your budget range?",
      "options": [
        { "label": "< $1k", "value": "lt_1k" },
        { "label": "$1k - $5k", "value": "1k_5k" },
        { "label": "$5k - $10k", "value": "5k_10k" },
        { "label": "> $10k", "value": "gt_10k" }
      ]
    }
  ]
}
```

Each question has:
- `id` — short unique key (used in the response JSON)
- `text` — the question displayed to the user
- `options` — array of `{ label, value }` pairs
- `allowOther` — (optional, default `true`) show a free-text "Other" input

### Step 2: Encode the Config in the URL

Base64-encode the JSON config and append it as a URL hash fragment:

```
url = FORM_BASE_URL + "#" + base64(encodeURIComponent(JSON.stringify(config)))
```

### Step 3: Send One Message with a Web App Button

Send a single message with a `web_app` button that opens the form:

```json
{
  "action": "send",
  "channel": "telegram",
  "message": "I have a few questions before we proceed. Tap the button below to fill them out.",
  "buttons": [
    [
      {
        "text": "Fill out form",
        "web_app": { "url": "<encoded-url-from-step-2>" }
      }
    ]
  ]
}
```

This opens the Mini App panel inside Telegram. The user sees:
- Numbered questions with radio-button options
- Visual "selected" highlighting on chosen options
- An "Other" text input per question for free-text answers
- A native Submit button at the bottom

### Step 4: Receive the Response

When the user taps Submit, you receive **one single message** with all answers as JSON.
No intermediate callbacks — everything arrives at once.

The response format:

```json
{
  "type": {
    "question": "What type of project is this?",
    "value": "mobile",
    "label": "Mobile App"
  },
  "timeline": {
    "question": "What is your timeline?",
    "value": "this_month",
    "label": "This month"
  },
  "budget": {
    "question": "What is your budget range?",
    "value": "1k_5k",
    "label": "$1k - $5k"
  }
}
```

If the user typed a free-text "Other" answer, `value` and `label` will both be
their typed text (e.g., `"value": "End of March", "label": "End of March"`).

### Step 5: Confirm and Proceed

Send a brief summary and proceed with the task:

```
Here's what I got:
• Project type: **Mobile App**
• Timeline: **This month**
• Budget: **$1k - $5k**

Let me work on this now.
```

## Key Behaviors

- **All state is client-side.** Nothing is sent to the agent until the user taps Submit.
- **Options show a selected state.** Tapped options highlight with the Telegram theme color.
- **Users can change answers freely.** They can tap different options before submitting.
- **Validation happens in the form.** The Submit button won't send until all questions are answered.
- **The form respects Telegram themes.** Dark mode, light mode, and custom themes all work.

## Inline Buttons Fallback

For simple single-question interactions (yes/no, pick one option), you can still use
regular inline buttons with `callback_data`. Only use the Mini App form for 2+ questions.

```json
{
  "action": "send",
  "channel": "telegram",
  "message": "Should I proceed?",
  "buttons": [
    [
      { "text": "Yes", "callback_data": "confirm_yes" },
      { "text": "No", "callback_data": "confirm_no" }
    ]
  ]
}
```

## Edge Cases and Advanced Patterns

See [references/form-patterns.md](references/form-patterns.md) for:
- Disabling "Other" on specific questions
- Handling very long option lists
- Form config size limits
- What happens if the user dismisses the form without submitting
