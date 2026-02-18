# questions-form

An [OpenClaw](https://docs.openclaw.ai/) skill that teaches agents how to ask clarifying questions on Telegram using a **single interactive form message** — no roundtrip spam, no acknowledgement messages, just buttons that update in place.

## What it does

Instead of asking questions one at a time (and burning tokens per click), the agent sends a **single message** with all questions and answers as inline buttons. The user taps to select, the message updates live, and the model is only invoked with a real output when Submit is tapped.

```
📋 A few questions before we proceed — tap to answer, then Submit.

1️⃣ What type of project? → Mobile ✅
2️⃣ What is your timeline?
3️⃣ Budget range? → $1k–5k ✅

[Web App]  [✅ Mobile]  [API]
[Other ✏️]
[This week]  [This month]  [No rush]
[Other ✏️]
[< $1k]  [✅ $1k–5k]
[$5k–10k]  [> $10k]
[Other ✏️]
[✓ Submit]  [✗ Cancel]
```

## Key features

- **Single message form** — all questions in one Telegram message, updated in-place
- **No token waste** — button taps edit the message silently (`NO_REPLY`); model only responds on Submit
- **Live selection feedback** — tapped buttons show `✅` prefix instantly
- **"Other ✏️" free-text option** — every question includes a custom-text fallback
- **Change answers** — re-tap any button before submitting; previous selection replaced
- **Partial submission handling** — warning shown inline when questions are unanswered
- **Advanced patterns** — dependent questions, multi-select toggles, timeout handling

## Installation

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

## Structure

```
questions-form/
├── SKILL.md                        # Main skill — form protocol & button schemas
└── references/
    └── form-patterns.md            # Advanced patterns & edge cases
```

## Requirements

- [OpenClaw](https://docs.openclaw.ai/) with Telegram channel configured
- Inline buttons enabled (`capabilities.inlineButtons` set to `"dm"`, `"all"`, or `"allowlist"`)
- `editMessage` action enabled (`channels.telegram.actions.editMessage` not disabled)

## How it works (architecture)

```
User taps button
      │
      ▼
OpenClaw routes callback_data to model
      │
      ▼
Model records answer + edits form message in-place
      │
      ▼
Model replies NO_REPLY  ← no chat output, no extra messages
      │
  (repeat per tap)
      │
User taps Submit
      │
      ▼
Model reads form_state → proceeds with full task response
```
