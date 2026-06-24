# Contribution 86: Add a Discord webhook notifier for delivery milestones (alongside Telegram)

**Contribution Number:** 86

**Student:** Luis C. Martinez

**Issue:** https://github.com/cesarnml/son-of-anton/issues/86#issue

**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because it is well-scoped, and it involves technologies that I currently use as an actual user like Discord. I want to try to use this tool in my personal development workflows.

---

## Understanding the Issue

### Problem Description
Currently The Son of Anton codebase currently supports milestone delivery notifications thrugh Telegram, but it doesn't support sending these same notifications to a Discord text channel. 

The goal of this issue is to add a Discord webhook notifier so delivery milestone messages send to a channel.

### Expected Behavior

Once a milestone delivery message such as ticket started," "PR opened," "review window ready," "ticket completed," are triggered then the codebase should send a text message to a discord channel that it is allowed to write to. 

### Current Behavior

Currently this behavior is non existing, but has a similar behavior implementation for a Telegram message. 

### Affected Components
Below is the affected file and functions that are touch by the telegram notification logic.
notifications.ts
orchestrator.test.ts

resolveNotifier()
notifyBestEffort()

## Reproduction Process
I am reproducing a telegram message to make sure that logic is working since the new discord webhook logic will live along the telegram message.

### Environment Setup

Easy install with bun.

### Steps to Reproduce

1. [Step 1 - create an env file] 
```
cp .env.example .env
```
2. [Step 2 - add telegram token and chat id required] 
```
TELEGRAM_BOT_TOKEN=... TELEGRAM_CHAT_ID=...
```
3. [Step 3 - export env vars & use notification methods to trigger telegram message] 
```
set -a
source .env
set +a

bun -e "import { notifyBestEffort, resolveNotifier } from './tools/delivery/notifications.ts'; const warning = await notifyBestEffort(resolveNotifier(), process.cwd(), { kind: 'run_blocked', planKey: 'telegram-smoke-test', command: 'manual-smoke', reason: 'Manual Telegram notification smoke test' }); if (warning) { console.warn(warning); process.exit(1); } console.log('Telegram notification sent.');"
```
4. [Observed result] - No warning. I have recieved a telegram message.

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

Currently there is no logic for a Discord webhook message. Most of the logic is filled out but there is a cveat because the currently Telegram implementation does not use markdowns.
Telegram messages currently use Telegram-specific formatting, including message entites for lins. Discord webhooks do not use Telegram-style entities. Instead Discord messages can include links directly in the message content using markdown syntax. Because of these differences the solution needs to make sure the message formatting is handled correctly for each platform. 

### Proposed Solution

To add logic so that each platform formats its final message in its own way. 

### Implementation Plan

Using framework (adapted):

**Understand:** The Problem
There currently is no Discord feature.

Match:
The existing Telegram notifier provides the closest pattern in the codebase. I will use it to understand:

How notifier configuration is resolved
How notification events are converted into messages
How best-effort notification failures are handled
How tests currently verify notification behavior

The Discord notifier should follow the same general shape, but it should use Discord's webhook API and Markdown-style links instead of Telegram entities. Following option (b), I will also adjust where the message is built so the links come out as structured data that both platforms can format.

Plan: [Step-by-step implementation plan]
Modify tools/delivery/notifications.ts.
Extend the DeliveryNotifier type with a Discord option that carries the webhook URL.
Update resolveNotifier() to return a Discord notifier when DISCORD_WEBHOOK_URL is configured, and document the precedence when both Telegram and Discord vars are set (simplest: Telegram wins).
Refactor message building so links are carried as structured data instead of being baked into a Telegram-only format (option b). The payload keeps the message text plus a list of links (label + url) that hasn't been formatted for any one platform yet.
Update the Telegram path so it builds its text + entities from that structured data, exactly matching current Telegram behavior so nothing regresses.
Add a sendDiscordMessage() function that formats the same structured links into Markdown and sends:
{
  "content": "..."
}
to the configured webhook URL.

Update notifyBestEffort() to call the correct sending function based on notifier.kind.
Add tests for:
Discord notifier resolution
Discord webhook payload format (the structured links render as Markdown [label](url) in content)
Best-effort failure handling (a mocked fetch failure becomes a warning, never a throw)
Existing Telegram behavior, to ensure it does not regress (entities still build correctly from the structured links)
Update .env.example with:
DISCORD_WEBHOOK_URL=
Update documentation to explain how to configure Discord notifications.
Run the project checks:
bun run format
bun run ci

Implement: 

Branch: https://github.com/luiscmartinez/son-of-anton/tree/fix-issue-86-discord-webhook
Commits: 
Pull Request:

Review: [Self-review checklist - does it follow the project's contribution guidelines?]
Discord notifier follows the same style as the existing Telegram notifier.
Message wording is built once and shared by both platforms (option b), so nothing is duplicated.
Telegram behavior still works (entities rebuilt from the structured links, no regression).
Discord webhook requests use the correct JSON payload with Markdown links.
Notification failures remain best-effort and do not crash the delivery process.
Tests cover the new Discord behavior and the Telegram regression.
Documentation and .env.example are updated.
Formatting and CI pass.

Evaluate: [How will you verify it works?]
I will verify the solution in three ways:

Automated tests

Run:

bun run format
bun run ci
Manual Discord smoke test

Configure:

DISCORD_WEBHOOK_URL=...

Then trigger a test notification and confirm that the message appears in the Discord channel with a working Markdown link.

Regression check

Re-run the Telegram smoke test to confirm that the existing Telegram notifier still works after adding Discord support.
---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
