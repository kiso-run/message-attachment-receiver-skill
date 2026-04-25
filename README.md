# message-attachment-receiver

A Kiso skill that turns user-uploaded attachments — audio,
images, documents — into the user's actual request, via two
stages: read each file with the right MCP, then re-plan with the
extracted content.

## When it fires

The skill activates when the user's message references uploaded
files via the `[Uploaded files: ...]` marker. The typical source
is a connector that downloads attachments into the session's
`uploads/` directory — for example the Discord connector at
`kiso-run/discord-connector` does exactly that.

## What it does

It only adds planner-side guidance. Each uploaded file is
classified by extension and routed to one specific MCP:

| Category   | Extensions                                  | MCP                                  |
|------------|---------------------------------------------|--------------------------------------|
| audio      | `.mp3 .m4a .wav .ogg .opus .webm .flac .aac` | `kiso-transcriber:transcribe_audio` |
| image      | `.png .jpg .jpeg .webp .gif .bmp .tiff .tif` | `kiso-ocr:ocr_image` (default) or `kiso-ocr:describe_image` |
| document   | `.pdf .docx .xlsx .csv .txt .md`            | `kiso-docreader:read_document`       |

The planner then emits a strict two-stage plan:

1. **Read stage** — one `mcp` task per attachment, all under a
   single `group` for parallel dispatch. Mixed types fan out
   together (one audio + one image + one PDF = three parallel
   `mcp` tasks).
2. **Replan** — the next planner pass receives the extracted
   text(s) and treats them as the user's real message. Intent is
   re-classified on the actual content.

## Why a skill, not an MCP

There is no new capability to expose — `kiso-transcriber`,
`kiso-ocr`, `kiso-docreader` already do the I/O. What was missing
was a deterministic rule for *when* and *how* to use them on
attachment input. That's prompt guidance, which is what skills
are for.

The replan loop is essential. The extracted content is unknown
at planning time, so the planner cannot decide the follow-up
action in the same plan as the read. A `replan` task between the
two is the standard Kiso primitive for "the next action depends
on the previous task's output".

## Install

```sh
kiso skill install --from-url \
    git+https://github.com/kiso-run/message-attachment-receiver-skill@v0.2.0
```

Or from a local Kiso checkout:

```sh
kiso skill install --from-url ./plugins/message-attachment-receiver-skill
```

The trust prefix is hardcoded in `kiso/core` (`SKILL_TIER1_PREFIXES`),
so the install is silent — no per-install consent prompt.

## Dependencies

The skill is a single `SKILL.md` with planner + reviewer
sections. At runtime it expects the appropriate MCP server to
be installed for each attachment type:

- `kiso-run/transcriber-mcp` for audio
- `kiso-run/ocr-mcp` for images
- `kiso-run/docreader-mcp` for documents

All three are in the default Kiso preset. If a needed server is
missing, the planner emits a `msg` task explaining how to
install it — never a silent fallback to exec, which would
produce wrong content.

## Limits

- File size limits and supported codecs/formats are inherited
  from each upstream MCP. The planner replans to a `msg` if the
  read fails (for example, a 50 MB audio that exceeds the
  transcriber's compression budget).
- Extensions are matched case-insensitively but only exactly:
  `.tar.gz` won't match anything; the user gets the default
  planner behaviour (`exec cat` / `exec head`).

## Version history

- **v0.2.0** — generalised from voice-only to all attachment
  types (audio, image, document), multi-file with parallel
  group dispatch.
- **v0.1.0** — voice-only (`voice-message-receiver`),
  audio-to-transcript via `kiso-transcriber`.
