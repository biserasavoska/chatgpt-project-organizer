# ChatGPT Project Organizer

An **unofficial, browser-console runbook** for auditing a large ChatGPT history, classifying conversations by their actual content, and moving them into ChatGPT Projects in bulk.

This repository documents the workflow that was actually used successfully on a large account with thousands of chats. It deliberately includes the browser-console techniques, undocumented web-app endpoints, `gizmo_id` behavior, custom-GPT handling, and troubleshooting that made the cleanup work.

## What this does

The workflow lets you:

- enumerate non-archived ChatGPT conversations;
- distinguish chats already in Projects from truly uncategorized chats;
- distinguish **custom-GPT chats** from Project chats;
- fetch the full contents of conversations for classification;
- download JSON inventories for review with ChatGPT or another LLM;
- bulk-move approved chats into Projects;
- retry or isolate inaccessible/legacy chats;
- avoid rescanning the entire account when you only need to move a visible sidebar chat;
- run a final audit.

## Important warning

This is **not an official OpenAI API**. It uses the same undocumented web-app endpoints the ChatGPT site itself uses. These endpoints can change without notice.

Use this only on an account you own or are authorized to manage.

Never paste, print, save, or commit your ChatGPT access token or cookies. The scripts obtain the current session token in-memory from `/api/auth/session` and do not intentionally print it.

Respect rate limits. If you receive `429 Too Many Requests`, stop and wait rather than trying to bypass the limit.

## The workflow in one page

1. **Discover your Project IDs** with `scripts/00-discover-projects.js`.
2. Put those IDs into the config section of the audit/export scripts.
3. Run `scripts/01-audit.js` to understand the account.
4. Run `scripts/02-export-uncategorized-full.js` to download the full contents of uncategorized chats.
5. Classify the JSON using `prompts/CLASSIFY.md`.
6. Review the proposed destinations before changing anything.
7. Put approved assignments into `scripts/04-run-moves.js` and run it in batches.
8. Re-audit.
9. Inspect generic leftovers by **actual content**, not title.
10. Run `scripts/03-export-custom-gpt-full.js` to capture custom-GPT conversations.
11. Classify those too.
12. Test moving **one** custom-GPT conversation first.
13. If successful, bulk-move the rest.
14. Use `scripts/05-move-visible-sidebar-chats.js` for stubborn visible chats without rescanning the whole history.
15. Final audit.

See [`docs/WORKFLOW.md`](docs/WORKFLOW.md) for the detailed version.

## Why classification must use message content

The biggest lesson from the cleanup was that titles are unreliable.

Chats titled things like:

- `New chat`
- `Conversation Summary Request`
- `Document Summary Request`
- `User query incomplete`

can contain substantial research, client work, recipes, technical troubleshooting, business ideas, or multi-month working threads.

Do **not** archive or classify generic titles without reading their actual messages.

## Project vs custom GPT: the important `gizmo_id` distinction

A conversation with a non-null `gizmo_id` is **not necessarily in a Project**.

In practice:

- Project IDs looked like `g-p-...`
- custom GPT IDs looked like `g-...`

The reliable method is to maintain a set of **your known Project IDs** and classify every other non-null `gizmo_id` as a custom-GPT/other-gizmo conversation.

This distinction was essential. An early audit that treated every `gizmo_id` as "already categorized" missed a large collection of custom-GPT chats.

## Tested endpoint pattern

Conversation listing:

```text
GET /backend-api/conversations?offset=<n>&limit=28&order=updated&is_archived=false
```

Conversation fetch:

```text
GET /backend-api/conversation/<conversation_id>
```

Move to Project:

```text
PATCH /backend-api/conversation/<conversation_id>
Content-Type: application/json

{"gizmo_id":"g-p-..."}
```

The scripts also send the web-app headers that proved necessary in testing.

## Tested failure modes

The workflow encountered and handled:

- `404 conversation_inaccessible`
- `500 {"detail":"Something went wrong."}` on a legacy/special conversation
- `429 Too Many Requests` during long scans
- stale sidebar state after successful moves
- duplicate/generic titles
- custom-GPT chats masquerading as "categorized" because they also have a `gizmo_id`
- conversations whose useful current ID only became visible on a later audit

See [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md).

## Files

```text
scripts/
  00-discover-projects.js
  01-audit.js
  02-export-uncategorized-full.js
  03-export-custom-gpt-full.js
  04-run-moves.js
  05-move-visible-sidebar-chats.js
  06-retry-single-chat.js

prompts/
  CLASSIFY.md
  REVIEW-LEFTOVERS.md

docs/
  WORKFLOW.md
  WHAT-WORKED.md
  TROUBLESHOOTING.md
  SECURITY.md

templates/
  projects.example.json
  assignments.example.js
```

## Browser setup

Use ChatGPT in a desktop browser while logged into the account you are organizing.

Open Developer Tools → Console.

On Chrome/macOS:

```text
Option + Command + J
```

Paste one script at a time.

## Do not commit personal data

The exported JSON contains the full text of your conversations. Keep it out of Git.

The included `.gitignore` excludes the filenames produced by these scripts.

## License

MIT. See [`LICENSE`](LICENSE).
