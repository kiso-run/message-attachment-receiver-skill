---
name: voice-message-receiver
description: Transcribe an uploaded voice message first, then re-process the transcript as the user's actual message.
when_to_use: User attaches an audio file (mp3, m4a, wav, ogg, opus, webm, flac, aac) — typically via a Discord/Telegram-like connector that drops voice notes into the session uploads/ directory.
activation_hints:
  applies_to: ["audio", "voice"]
version: 0.1.0
---

## Planner

Trigger: the `## New Message` user content includes
`[Uploaded files: ...]` referencing a file whose extension is one of
`.mp3 .m4a .wav .ogg .opus .webm .flac .aac` (case-insensitive). The
upload lives at `uploads/<filename>` under the session workspace.

When this trigger fires, **emit a two-stage plan**:

1. A single `mcp` task targeting `kiso-transcriber:transcribe_audio`
   with `args.file_path` set to the workspace-relative path
   (`uploads/<filename>`). `expect`: a non-empty transcript text.
   If the user's message also includes a language hint (e.g.
   *"in italiano"*, *"in english"*), pass it through
   `args.language` as the ISO 639-1 code (`it`, `en`, ...).
2. A final `replan` task with detail
   *"The previous task transcribed the user's voice message.
   Treat the transcript as the user's actual request and re-plan
   from scratch — classify it, route it, plan accordingly. Do not
   transcribe again."*

Rules — keep these strict:

- **Two stages only.** Never put the transcription and the actual
  follow-up action in the same plan. The transcript is unknown
  content at planning time; the planner needs the replan loop to
  re-evaluate intent (chat / plan / investigate / chat_kb).
- **Audio is for transcriber only.** Do not route the audio file
  through `kiso-ocr`, `kiso-docreader`, or any other MCP — the
  transcription step is the canonical reading.
- **Multiple uploads in one message:** if both audio AND non-audio
  files are uploaded in the same message, transcribe the audio
  first (this skill's job) then replan; the next plan handles the
  other files normally. If multiple audio files are uploaded,
  transcribe them in parallel using the `group` field on each mcp
  task, then a single replan.
- **No transcription on text uploads.** The skill is silent if the
  upload set contains zero audio files — fall through to the
  default planner behaviour.

## Reviewer

When reviewing the transcription stage, accept on:
- exit `ok` with non-empty `output` containing the transcript text.
Reject (replan) on:
- empty output (silent or unintelligible audio — the planner will
  msg the user).
- explicit `is_error` from the MCP transport.
