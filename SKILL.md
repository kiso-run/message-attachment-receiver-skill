---
name: message-attachment-receiver
description: "Read uploaded attachments (audio, images, documents) via the right MCP, then re-plan with their content as if the user had typed it inline."
when_to_use: "User attaches files to a message — typically via a Discord/Telegram-like connector that drops attachments into the session uploads/ directory and appends an Uploaded files marker to the chat content."
activation_hints:
  applies_to: ["audio", "voice", "image", "document", "attachment"]
version: 0.2.0
---

## Planner

Trigger: the `## New Message` user content includes
`[Uploaded files: <list>]`. Each filename's extension classifies
it into exactly one of three categories:

| Category   | Extensions (case-insensitive)                              | MCP target                          |
|------------|------------------------------------------------------------|-------------------------------------|
| `audio`    | `.mp3 .m4a .wav .ogg .opus .webm .flac .aac`               | `kiso-transcriber:transcribe_audio` |
| `image`    | `.png .jpg .jpeg .webp .gif .bmp .tiff .tif`               | `kiso-ocr:ocr_image` (default) or `kiso-ocr:describe_image` (when the user asks about visual content rather than text) |
| `document` | `.pdf .docx .xlsx .csv .txt .md`                           | `kiso-docreader:read_document`      |

When this trigger fires, **emit a two-stage plan**:

1. **Read stage** — one `mcp` task per attachment. Each task
   targets the MCP method from the table above, with
   `args.file_path` set to the workspace-relative path
   (`uploads/<filename>`). Optional args:
   - audio: `args.language` if the user gave a language hint
     (`it`, `en`, ...).
   - image: choose `ocr_image` by default; switch to
     `describe_image` when the user's prompt is about visual
     content (*"cosa si vede"*, *"descrivimi"*, *"che foto è"*).
   - document: `args.pages` if the user asked for a page range.

   `expect`: a non-empty extracted text or description.

   When **two or more attachments** are present, group all
   read-stage tasks under the same `group` integer (e.g. `group:
   1`) so the worker dispatches them in parallel. A single
   attachment doesn't need a group field.

2. **Replan task** with detail
   *"The previous task(s) read the user's uploaded
   attachment(s). Treat the extracted text as the user's
   actual request and re-plan from scratch — classify it,
   route it, plan accordingly. Do not re-read the
   attachments."*

Rules — keep these strict:

- **Two stages only.** Never put the read tasks and the
  follow-up action in the same plan. The extracted content is
  unknown at planning time; the planner needs the replan loop
  to re-evaluate intent (chat / plan / investigate / chat_kb)
  on the actual content.
- **One MCP per attachment, no cross-routing.** Audio goes only
  to `kiso-transcriber`; never to OCR or docreader. Images go
  only to `kiso-ocr`. Documents only to `kiso-docreader`. Never
  ad-hoc `exec ffmpeg` or `exec cat`.
- **Multi-file = parallel group.** Use the `group` field on
  every read task in the same plan, so the worker can fire
  them concurrently. The replan task is NOT in the group (it
  must run after the reads complete).
- **Mixed-tier attachments fan out together.** If the user
  uploads one audio + one image + one PDF in the same message,
  emit three `mcp` tasks in a single group, then one `replan`.
- **Mismatched MCP availability.** If the briefer's `## MCP
  Methods` block does not list the target server for one of
  the categories present, emit a `msg` task that names the
  missing capability and tells the user how to install it
  (`kiso mcp install --from-url ...`). Do NOT fall back to
  exec — the wrong tool would silently produce wrong content.
- **No-op fallback.** If the upload set contains zero
  recognised extensions, this skill is silent — fall through
  to default planner behaviour (`upload_hint`, exec read of
  arbitrary files).

## Reviewer

For each read-stage task:
- Accept (`ok`) on exit `ok` with non-empty `output`. The
  `output` is the extracted text and is the canonical form
  the next planner pass will see.
- Reject (`replan`) on empty output (silent audio, blank
  image with no detectable text, completely empty document)
  or on explicit `is_error` from the MCP transport. The next
  plan will typically `msg` the user explaining the problem
  and asking for a clearer file.
