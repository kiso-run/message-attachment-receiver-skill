# voice-message-receiver

A Kiso skill that turns an uploaded voice message into the user's
actual request via two stages: transcribe → re-plan.

## When it fires

The skill activates when the user's message references an uploaded
audio file (`.mp3 .m4a .wav .ogg .opus .webm .flac .aac`). The
typical source is a connector that downloads voice notes into the
session's `uploads/` directory and appends `[Uploaded files: ...]`
to the chat content — for example the Discord connector at
`plugins/discord-connector` does exactly that.

## What it does

It only adds planner-side guidance. The skill body extends the
planner prompt with:

1. Plan stage 1 — `mcp` call to `kiso-transcriber:transcribe_audio`
   with the workspace-relative audio path.
2. Plan stage 2 — `replan` so the next planner pass sees the
   transcript and treats it as the user's actual message.

Rationale:

- The transcript is unknown at planning time. Letting the planner
  do everything in one shot would force it to predict the user's
  intent before reading the audio. The replan loop is exactly the
  primitive Kiso uses elsewhere when the next action depends on a
  prior task's output.
- The transcriber is single-purpose. Routing audio through it
  rather than letting the planner improvise (`exec ffmpeg`,
  generic OCR, etc.) keeps the workflow deterministic.

## Install

```sh
kiso skill install --from-url git+https://github.com/kiso-run/voice-message-receiver-skill@v0.1.0
```

Or from a local Kiso checkout:

```sh
kiso skill install --from-url ./plugins/voice-message-receiver-skill
```

## Dependencies

The skill itself is a single `SKILL.md` (planner-only guidance).
At runtime it expects `kiso-run/transcriber-mcp` to be installed
and reachable in the session's MCP catalog — the default Kiso
preset already includes it. Without `kiso-transcriber:transcribe_audio`
the planner will fall back to its standard upload-handling rules
and the skill becomes a no-op.

## Limits

- Audio decoding is delegated to `kiso-transcriber-mcp`; supported
  formats are whatever that server supports (Gemini multimodal +
  ffmpeg compression).
- Files larger than the transcriber's hard limit (currently 20 MB
  before compression in `transcriber-mcp`) will fail at MCP call
  time with a clear error; the planner will replan to a `msg`
  asking the user for a smaller file.
