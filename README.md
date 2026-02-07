# questions-form

An [OpenClaw](https://docs.openclaw.ai/) skill that teaches agents how to ask clarifying questions on Telegram using interactive inline-button forms.

## What it does

Instead of asking questions as plain text, the agent presents a structured form:

```
I have a few questions before we proceed.

1. What type of project?
[Web App] [Mobile] [API]
[Other (type your answer)]

2. What is your timeline?
[This week] [This month] [No rush]
[Other (type your answer)]

[✓ Submit All Answers]
[✗ Cancel]
```

Users tap buttons to answer, type free text for "Other", and submit when ready. The agent collects all answers before proceeding.

## Key features

- **Multi-question forms** — all questions shown at once, answered in any order
- **"Other" free-text option** — every question includes a fallback for custom input
- **Change answers** — users can re-tap buttons before submitting
- **Partial submission handling** — agent tells users which questions still need answers
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
