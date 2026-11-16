# Voicebox plugin for Claude Code

This plugin connects Claude Code to a self-hosted [Voicebox](https://github.com/jamiepine/voicebox) server at `voice.snapcoder.ai`. Claude can then speak text in a chosen voice, hand you the audio file, and transcribe recordings. A bundled skill tells Claude which engine suits which voice, language and style, and how to write text for each engine.

The server is private. You need an access key from its owner.

## Install

```
/plugin marketplace add 12northlabs/voicebox-claude-plugin
/plugin install voicebox@voicebox-plugins
```

Claude Code asks for the access key when the plugin is enabled and keeps it in your system's secure credential store.

## Tools

| Tool | What it does |
| ---- | ------------ |
| `list_voices` | Lists the voice profiles with their type, language and engines. |
| `speak` | Speaks text in a voice and returns an MP3 or WAV download link that works for 24 hours. |
| `transcribe` | Transcribes base64-encoded audio, up to 10 MB, with Whisper. |
| `gpu_status` | Shows which voice engines and language models hold GPU memory. |
| `unload_voice_engines` | Frees the GPU after a batch of speech work. |

Voice cloning is not available through the plugin.

## claude.ai

The same server works as a claude.ai custom connector. Add `https://voice.snapcoder.ai/mcp?key=<your key>` as the connector URL. The skill is not loaded there, so the tool descriptions are all Claude sees.

## Licence

MIT. Voicebox and its speech models have their own licences.

## Privacy

The text you ask Claude to speak, and the audio you send for transcription, go to the server's owner. Voicebox keeps a history of generations on that server. Do not send anything you would not share with the owner.
