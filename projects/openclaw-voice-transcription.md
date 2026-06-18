# OpenClaw Voice Transcription

Field note for configuring local inbound voice-message transcription in an OpenClaw runtime.

## Pattern

OpenClaw can use a local Whisper-compatible command for audio transcription. A practical low-resource setup is to expose a small `whisper` wrapper on the workspace path and back it with [`faster-whisper`](https://github.com/SYSTRAN/faster-whisper) in a local virtual environment.

The wrapper should preserve enough of the common Whisper CLI shape for OpenClaw media tooling:

- accept `--model`
- accept optional `--language`
- accept `--output_format`
- accept `--output_dir`
- accept and ignore compatibility flags that the local backend does not need, such as `--task transcribe`
- print the transcript to stdout
- write the transcript file when an output directory is requested

## Runtime choices

Use CPU `int8` inference for a modest default on small gateway hosts. The `tiny` model is a reasonable first validation target because it keeps cold-start and memory cost low. Larger models can be tested later if accuracy becomes more important than latency and resource use.

Keep speech language auto-detected when users commonly send voice messages in more than one language. Pinning a language can improve accuracy for a known single-language channel, but it is the wrong default for mixed-language DMs.

## Public snapshot hygiene

Commit the small wrapper scripts if they are public-safe and useful for reproducing the workspace setup. Do not commit the virtual environment, downloaded models, audio files, transcripts from private conversations, or runtime caches.

For Se7en's blueprint snapshot, `.venvs/` is excluded and the public-safe `bin/whisper*` wrappers are allowed.

## Validation

A minimal validation path is:

1. Configure OpenClaw media audio settings to call the wrapper.
2. Send a short voice message through the target channel.
3. Confirm the inbound message arrives with a transcript.
4. Confirm mixed-language expectations before forcing a `--language` value.
5. Review any public notes for private transcript content before committing.
