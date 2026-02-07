# Form Patterns Reference

Advanced patterns and edge cases for the questions-form Mini App skill.

## Disabling "Other" on Specific Questions

By default every question shows an "Other — type your answer" text input.
To disable it for a specific question, set `allowOther: false`:

```json
{
  "id": "priority",
  "text": "Priority level?",
  "allowOther": false,
  "options": [
    { "label": "Low", "value": "low" },
    { "label": "Medium", "value": "medium" },
    { "label": "High", "value": "high" }
  ]
}
```

## Long Option Lists

The form renders options as a vertical list of tappable rows, so many options
work well. For questions with 8+ options, consider grouping related choices
or splitting into two questions instead.

The form scrolls naturally on mobile — no special handling needed.

## Form Config Size Limits

The config is base64-encoded in the URL hash. Telegram Web App URLs have a
practical limit of ~2048 characters. This is enough for roughly 10-15 questions
with 4-5 options each.

If you exceed this:
- Shorten question text and option labels
- Use shorter `id` and `value` strings
- Split into multiple forms sent sequentially

## User Dismisses the Form

If the user swipes down or taps outside to close the Mini App without
submitting, **nothing is sent to the agent**. The agent receives no message.

How to handle this:
- The agent should not block indefinitely waiting for form data
- If the user sends a regular text message after you sent the form button,
  treat it as them choosing not to use the form — respond normally
- You can ask: "I sent you a form to fill out. Would you prefer to answer
  the questions here in chat instead?"

## Form Theming

The Mini App automatically uses Telegram's theme variables:
- `--tg-theme-bg-color` — background
- `--tg-theme-text-color` — text
- `--tg-theme-button-color` — selected option highlight & submit button
- `--tg-theme-secondary-bg-color` — unselected option background
- `--tg-theme-hint-color` — placeholder text & question numbers

This means the form matches the user's Telegram appearance (dark mode, custom
themes, etc.) with no extra configuration.

## Complete Example

### Agent builds the config:

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

### Agent encodes and sends:

```
base64(encodeURIComponent(JSON.stringify(config))) → "eyJxdWVzdGlvbn..."
url = "https://owner.github.io/skill-questions-form/form.html#eyJxdWVzdGlvbn..."
```

```json
{
  "action": "send",
  "channel": "telegram",
  "message": "I have a few questions before we proceed. Tap the button below to fill them out.",
  "buttons": [[{ "text": "Fill out form", "web_app": { "url": "<url>" } }]]
}
```

### User experience:

1. User taps "Fill out form" — Mini App panel slides up inside Telegram
2. User sees 3 questions with radio-button options and "Other" inputs
3. User taps options — they highlight with the theme color
4. User can change any answer by tapping a different option
5. User taps "Submit All Answers" (Telegram's native main button)
6. Mini App closes, form data sent to agent as one JSON message

### Agent receives:

```json
{
  "type": { "question": "What type of project is this?", "value": "mobile", "label": "Mobile App" },
  "timeline": { "question": "What is your timeline?", "value": "End of March", "label": "End of March" },
  "budget": { "question": "What is your budget range?", "value": "1k_5k", "label": "$1k - $5k" }
}
```

### Agent responds:

```
Here's what I got:
• Project type: **Mobile App**
• Timeline: **End of March**
• Budget: **$1k - $5k**

Let me work on this now.
```
