# Contribution 86: Add a Discord webhook notifier for delivery milestones (alongside Telegram)

**Contribution Number:** 86

**Student:** Luis C. Martinez

**Issue:** https://github.com/cesarnml/son-of-anton/issues/86#issue

**Status:** Phase IIII Complete

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

- **Commit showing reproduction:** No commit needed — reproducing this was about confirming the existing Telegram path works before adding Discord next to it. No code changes were required for that.
- **Screenshots/logs:** The script printed `Telegram notification sent.` with no warning, and the message showed up in my Telegram chat. [TODO: add screenshot of the Telegram message]
- **My findings:** The Telegram notifier works exactly like the issue describes. The important discovery was in `buildNotificationPayload`: only the two standalone review events actually attach a link entity. Every other event (PR opened, ticket completed, etc.) just puts the PR URL in the text as plain text. That mattered a lot later when deciding how to handle links for Discord.

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

(Note: this was my original plan. During implementation I ended up going with a third option (c) instead — reusing the existing `{ text, entities }` payload without refactoring it. See Implementation Notes for why.)

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
Commits: `d95d8e8` (feature), `5c157ef` (review fix)
Pull Request: https://github.com/cesarnml/son-of-anton/pull/104

Review: [Self-review checklist - does it follow the project's contribution guidelines?]
Discord notifier follows the same style as the existing Telegram notifier.
Message wording is built once and shared by both platforms (option c), so nothing is duplicated.
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

- [x] Test case 1: `resolveNotifier` resolution — returns the `discord` notifier when `DISCORD_WEBHOOK_URL` is set, `noop` when nothing is configured, and a whitespace-only webhook resolves to `noop` (exercises the `.trim()` guard).
- [x] Test case 2: Precedence — when both `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` and `DISCORD_WEBHOOK_URL` are set, Telegram wins and Discord is ignored.
- [x] Test case 3: `buildDiscordContent` formatting — splices `text_link` entities into Markdown `[label](url)` left-to-right (offset-stable across multiple entities), escapes inline Markdown (`` \ ` * _ ~ | [ ] ( ) ``) and line-start block markers (`#`, `>`, `-`/`+`, `N.`) so free-form text renders literally, and skips out-of-range entity offsets instead of emitting a broken link. Also pins the maintainer's Markdown-injection repro: `[click me](https://evil.example.com)` in a note gets escaped instead of becoming a live link.

### Integration Tests

- [x] Integration scenario 1: Full `notifyBestEffort` → `sendDiscordMessage` path with a mocked `fetch` — asserts the exact POST body (`content`, `flags: 4`, `allowed_mentions: { parse: [] }`) for a standalone review event (Markdown PR link) and a ticketed event (bare auto-linked URL), confirming only standalone events get a `[PR #N](url)` link.
- [x] Integration scenario 2: Best-effort failure handling — a mocked non-2xx Discord response is swallowed into a `"Notification warning: Discord webhook failed with 500"` string and never throws or aborts delivery. The two pre-existing Telegram tests were also hardened to isolate `DISCORD_WEBHOOK_URL` (no cross-test env leakage).

### Manual Testing

Automated: `bun run format` then `bun run ci` pass locally. At the time I opened the PR the suite was **628 tests / 0 failures**; after upstream merges and the review fix it grew to **666 tests / 0 failures**. The Discord HTTP layer is covered via a mocked `globalThis.fetch`, asserting the literal request payload rather than hitting Discord. The maintainer also verified the test suite locally as part of his review.

I also ran the Telegram smoke test from the reproduction section again after the change to confirm the existing notifier still works.

**Live Discord test (June 30):** I pointed `DISCORD_WEBHOOK_URL` at a webhook in my own server's #bot-testing-channel and ran the orchestrator for real. Two genuine `run_blocked` milestone notifications came through (one from a failed `triage-standalone`, one from a usage error). This was a better test than the issue's curl example because the messages were produced by the actual delivery pipeline — and the usage message is full of brackets and angle brackets (`[--pr <number>]`, `[ticket-id]`, etc.) that all rendered as literal plain text, with no link previews and no pings.

![Discord live test — two run_blocked notifications in #bot-testing-channel](./discord-live-test.png)

---

## Implementation Notes

### Week 1 Progress (June 18)

Claimed the issue and got assigned. Read through `tools/delivery/notifications.ts` and mapped how the Telegram notifier works (resolveNotifier, the event → message formatting, the best-effort send). Set up the repo with bun and ran the Telegram smoke test to reproduce the existing behavior before touching anything.

### Week 2 Progress (June 30)

The big decision this week was link formatting. My original plan was option (b) from the issue (refactor the payload into structured `{ label, url }` links). After stress-testing that plan against the actual code, I realized only the two standalone review events carry a link today, and option (b) would rewrite exactly the offset math that the "no Telegram regression" requirement is protecting. So I went with option (c) instead: keep the `{ text, entities }` payload untouched and have Discord render from the same payload. Implemented the notifier, tests, and docs, and opened PR #104.

A challenge I didn't expect: Discord renders `content` as Markdown but Telegram renders plain text, so a ticket title like `# Rework auth` would turn into a giant heading on Discord. I ended up writing a line-aware escaper (inline markers everywhere, block markers like `#`/`>`/`-` only at line start) so text reads the same on both platforms.

### Week 3 Progress (July 2–5)

The maintainer reviewed on July 2. Everything passed except one thing: I wasn't escaping `[`, `]`, `(`, `)`, so a `[text](url)` pattern in a free-form note would render as a real clickable link on Discord. I had actually considered this and skipped it to avoid visible backslashes, but his framing was better than mine — the note field can come from AI review comments (lower-trust text), so it's an injection vector, not just cosmetics. Fixed it on July 4 (`5c157ef`), added a test with his exact repro string, and replied on the PR. Approved and merged July 5.

### Code Changes

- **Files modified:** `tools/delivery/notifications.ts`, `tools/delivery/test/orchestrator.test.ts`, `.env.example`, `docs/template/delivery/delivery-orchestrator.md`
- **Key commits:**
  - [d95d8e8](https://github.com/cesarnml/son-of-anton/pull/104/commits/d95d8e8287731966956e29888fbd161a4565c2fb) — feat(notifications): add Discord webhook notifier
  - [5c157ef](https://github.com/cesarnml/son-of-anton/pull/104/commits/5c157ef) — fix(notifications): escape brackets/parens to block Markdown-link injection
- **Approach decisions:** 
I chose option (c) — render Discord from the existing { text, entities } payload — because it leaves the Telegram path byte-for-byte unchanged (the lowest-risk way to guarantee no regression) while still building the message wording only once. The supporting calls follow from that: Telegram-wins precedence preserves existing users' behavior, and per-line Markdown escaping + allowed_mentions: { parse: [] } + flags: 4 make Discord render free-form text literally and safely, matching Telegram's plain-text output.
---

## Pull Request

**PR Link:**  https://github.com/cesarnml/son-of-anton/pull/104

**PR Description:**
Adds Discord as a second delivery-milestone notification channel alongside Telegram, implementing #86. The milestone events themselves are unchanged — this only adds a new delivery channel.


**Maintainer Feedback:**
- **July 2:** cesarnml reviewed (changes requested). Overall positive — he verified the tests locally, confirmed all acceptance criteria were met, and called the option (c) design "a well-reasoned deviation" from the issue's suggestions. One fix requested before merge: `escapeDiscordMarkdown` didn't escape `[`, `]`, `(`, `)`, so a `[click me](https://evil.example.com)` pattern in free-form text rendered as a real clickable link in Discord. The risky path is `review_recorded`'s note field, which can come from AI review comments (lower-trust content) — so it's a Markdown-link injection issue, not just formatting.
- **July 4:** Addressed it by adding `[]()`  to the inline escape set, with a test asserting his exact repro string gets escaped. Pushed `5c157ef` and replied on the PR explaining the fix and confirming the intentional entity-driven links still work.
- **July 5:** Approved and merged.

**Status:** Merged

---

## Learnings & Reflections

### Technical Skills Gained

Discriminated unions in TypeScript — adding a kind: 'discord' variant and letting kind-based narrowing drive safe dispatch in notifyBestEffort.
Two link models, one payload — Telegram uses offset/length entities over plain text; Discord uses Markdown. Designing one neutral payload that both render from avoids duplicating message wording.
Markdown escaping nuance — inline markers (* _ ` ~ |) fire anywhere, but block markers (#, >, -, N.) only at line start, so escaping has to be line-aware. I originally skipped escaping []() to avoid visible backslashes, but the maintainer's review changed my mind: when text can come from a lower-trust source, an unescaped [text](url) is a link injection vector. Threat model beats cosmetics.
Mention safety is a config, not an escape — backslashing @ doesn't stop Discord pings; allowed_mentions: { parse: [] } is the correct guard against accidental @everyone.
Best-effort side effects — a notification failure must degrade to a warning, never throw or abort the delivery run.
Test hygiene — asserting exact HTTP payloads via a mocked fetch, and avoiding the process.env.X = undefined → string "undefined" coercion trap when saving/restoring env vars.


### Challenges Overcome

- **Picking the right link-formatting option.** The issue offered (a) and (b), and my plan said (b). What got me unstuck was reading the actual code instead of trusting the issue's framing: only two event kinds carry a link today, so option (b) meant rewriting tested offset math for almost no payoff. Going with (c) kept the Telegram path completely untouched, which was the safest way to meet the "no regression" requirement.
- **Two platforms, two rendering models.** Telegram treats the message as plain text with link entities on the side. Discord treats the whole message as Markdown. Making the same text look identical on both meant writing an escaper, and getting the escaper right was harder than expected (line-start block markers vs inline markers, and not escaping the intentional links).
- **A sneaky test env trap.** Restoring env vars with `process.env.X = original` writes the literal string "undefined" when the var was never set, which is truthy and would silently flip tests to the wrong notifier. Wrote a small setOrDeleteEnv helper and used it everywhere.
- **Working on a moving branch.** The remote branch picked up commits from parallel work while I was building, including a duplicate of one of my own commits. Solved it by rebasing my feature commit on top of the remote instead of force-pushing over someone else's work.
- **Taking review feedback well.** The maintainer's requested change was something I had considered and talked myself out of. His reasoning (untrusted input → injection) was better than mine (ugly backslashes). Good reminder that a deliberate decision can still be the wrong one.

### What I'd Do Differently Next Time

[Reflection on your process]
I still need to spend more time with the code. I have rush thru implementation. 
---

## Resources Used

- [Discord webhook docs](https://discord.com/developers/docs/resources/webhook) — the execute-webhook endpoint, `flags` (SUPPRESS_EMBEDS), and `allowed_mentions`
- [Telegram Bot API — sendMessage](https://core.telegram.org/bots/api#sendmessage) — how `entities` / `text_link` work, for understanding the existing notifier
- [Issue #86](https://github.com/cesarnml/son-of-anton/issues/86) — the issue itself was the best resource; it maps the file, the functions, and calls out the link-formatting gotcha up front
- [PR #104 review feedback](https://github.com/cesarnml/son-of-anton/pull/104) — the maintainer's injection example taught me more about Markdown escaping than the docs did
- `docs/how-son-of-anton-works.md` in the repo — high-level picture of how delivery emits milestone events
