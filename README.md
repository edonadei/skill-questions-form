# questions-form

An [OpenClaw](https://docs.openclaw.ai/) skill that teaches agents how to ask clarifying questions on Telegram using a **Mini App form** that opens natively inside the chat.

## What it does

Instead of asking questions as plain text (or sending individual inline buttons that each trigger the agent), the agent opens an interactive form panel inside Telegram:

- All questions with radio-button options and "Other" free-text inputs
- Visual selected states on chosen options
- Everything stays **client-side** until the user taps Submit
- The agent receives all answers at once as a single JSON message

```
┌──────────────────────┐
│  1. Project type?     │
│  ● Web App            │  ← highlighted
│  ○ Mobile App         │
│  ○ API / Backend      │
│  Other: [________]    │
│                       │
│  2. Timeline?         │
│  ○ This week          │
│  ● This month         │  ← highlighted
│  ○ No rush            │
│  Other: [________]    │
│                       │
│  [Submit All Answers] │
└──────────────────────┘
```

## Setup

### 1. Deploy the Mini App

The form HTML needs to be served over HTTPS. The easiest way is GitHub Pages:

1. Fork or clone this repo
2. Go to **Settings > Pages** in your GitHub repo
3. Set source to **Deploy from a branch**, branch `main`, folder `/docs`
4. Your form will be available at `https://<username>.github.io/skill-questions-form/form.html`

### 2. Install the skill

Add the skill directory to your OpenClaw config (`openclaw.json`):

```json5
{
  "skills": {
    "load": {
      "extraDirs": ["/path/to/skill-questions-form"]
    }
  }
}
```

Or copy `questions-form/` into your managed skills directory.

### 3. Update the base URL

In `questions-form/SKILL.md`, update the `FORM_BASE_URL` to your deployed URL:

```
FORM_BASE_URL = https://<username>.github.io/skill-questions-form/form.html
```

## How it works

1. Agent builds a form config (questions + options as JSON)
2. Config is base64-encoded into the URL hash
3. Agent sends one message with a `web_app` button
4. User taps the button — Mini App panel slides up inside Telegram
5. User selects options, types "Other" answers, taps Submit
6. Agent receives all answers as one JSON payload and proceeds

## Structure

```
questions-form/
├── SKILL.md                        # Main skill — Mini App form protocol
└── references/
    └── form-patterns.md            # Advanced patterns & edge cases
docs/
└── form.html                       # Telegram Mini App (hosted via GitHub Pages)
```

## Requirements

- [OpenClaw](https://docs.openclaw.ai/) with Telegram channel configured
- The Mini App hosted on HTTPS (GitHub Pages, Vercel, or any static host)
