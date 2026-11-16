---
name: voicebox
description: Speak text aloud, make voice-overs or narration, and transcribe audio with the self-hosted Voicebox server through its MCP tools (list_voices, speak, transcribe, gpu_status, unload_voice_engines). Use when the user asks for speech, a voice, text-to-speech, an audio file of some text, or a transcript of a recording.
---

# Using Voicebox

Voicebox is a self-hosted text-to-speech and transcription server. Its MCP tools are `list_voices`, `speak`, `transcribe`, `gpu_status` and `unload_voice_engines`. The server shares one GPU with a large language model, so it can refuse work while that model is loaded.

## Workflow

1. Call `list_voices` once per session. Pick a voice from the names it returns; `speak` matches the name exactly.
2. Choose the engine from the voice's type (next section). For a preset or designed voice, leave `engine` empty.
3. Write the text for that engine (see "Writing the text").
4. Call `speak`. It waits for the audio and returns `url`, `duration_seconds`, `engine`, `voice`, `language` and `expires_unix`.
5. Save the audio for the user with `curl -fsSL -o <name>.mp3 '<url>'`. Quote the URL. The link works for 24 hours and needs no key.
6. For a transcript, read the file and pass it base64-encoded to `transcribe`, at most 10 MB decoded. In a shell, `base64 -w0 file.wav` gives the string. Set `language` when you know it.

## Voice types and engines

Each profile from `list_voices` has a `type`. The type decides which engines can speak it.

| Type | Engines that work | What to pass |
| ---- | ----------------- | ------------ |
| `preset` | only its `preset_engine` (`kokoro` or `qwen_custom_voice`) | no `engine` |
| `designed` | only `qwen_voice_design` | no `engine` |
| `cloned` | `qwen`, `luxtts`, `chatterbox`, `chatterbox_turbo`, `tada`, `omnivoice` | `engine` to choose; empty gives the profile's `default_engine`, else `qwen` |

A wrong pairing fails or sounds wrong. Never send `kokoro` with a cloned voice, or `qwen` with a preset.

Voice cloning is not available through these tools. If the user wants a new cloned voice, tell them to create it in the Voicebox web UI, then call `list_voices` again.

## Choosing an engine

| Need | Choose |
| ---- | ------ |
| Fast, clean English or other narration, no special emotion | a `Kokoro` preset. `Kokoro Heart` is the strongest English voice. |
| A named Qwen speaker, or emotion and style through `instruct` | a `Qwen` preset (`qwen_custom_voice`), size `1.7B` |
| Laughs, sighs or other sounds inside English text | a cloned voice with `chatterbox_turbo` |
| A language that no preset covers, such as Arabic | a cloned voice with `chatterbox` or `omnivoice` |
| The closest match to a cloned speaker in a supported language | a cloned voice with `qwen` (`1.7B`) |

Preset voices and their languages:

- Kokoro: English (`af_`/`am_` American, `bf_`/`bm_` British voices, shown as `Kokoro Heart`, `Kokoro Emma`, `Kokoro George` and so on), Spanish, French, Hindi, Italian, Portuguese, Japanese and Mandarin. The Mandarin Kokoro voices are weak; prefer a Qwen speaker for Chinese.
- Qwen CustomVoice: `Qwen Ryan` and `Qwen Aiden` (English), `Qwen Vivian`, `Qwen Serena`, `Qwen Uncle Fu`, `Qwen Dylan` and `Qwen Eric` (Chinese), `Qwen Ono Anna` (Japanese), `Qwen Sohee` (Korean). Any Qwen speaker can speak the other Qwen languages, with an accent.

Languages by engine:

- `qwen`, `qwen_custom_voice`, `qwen_voice_design`: zh, en, ja, ko, de, fr, ru, pt, es, it.
- `luxtts`, `chatterbox_turbo`: English only.
- `chatterbox`: 23 languages, including ar, da, de, el, es, fi, fr, he, hi, it, ja, ko, ms, nl, no, pl, pt, ru, sv, sw, tr and zh.
- `tada`: English at `1B`. `3B` adds ar, zh, de, es, fr, it, ja, pl and pt; its non-English quality is unverified.
- `omnivoice`: many languages, including Arabic.
- `kokoro`: en, es, fr, hi, it, pt, ja, zh.

Always pass `language` when the text is not English. The `language` must be one the engine speaks.

## Writing the text

- Write the words exactly as they should be heard. Spell out symbols, units and abbreviations that read badly ("3 km" becomes "three kilometres"; "e.g." becomes "for example").
- Use punctuation for pacing. A comma gives a short pause, a full stop a longer one, and a paragraph break a longer one still.
- Keep each call under 5,000 characters. For longer pieces, split at paragraph boundaries, call `speak` once per part with the same voice, engine and `seed`, and give the user the files in order.
- Kokoro sounds best with sentences of moderate length. Voicebox splits long text into chunks of about 800 characters by itself.
- Set `seed` to an integer when the user may want the same reading again.

### Emotion and style

- `instruct` is a plain-language direction, such as "speak slowly and warmly" or "excited, a little breathless". Only three engines read it:
  - `qwen_custom_voice` (the Qwen presets), best at `1.7B`.
  - `qwen_voice_design`, which adds it to the voice's design prompt.
  - `omnivoice`, which accepts only short attribute words such as "female, low pitch, british accent".
- Every other engine ignores `instruct`. For those, carry emotion with word choice and punctuation.
- Inline sound tags work only with two engines:
  - `chatterbox_turbo`: `[laugh]`, `[chuckle]` and `[cough]` are documented. Its tokenizer also lists `[sigh]`, `[gasp]`, `[clear throat]`, `[groan]`, `[sniff]` and `[shush]`, which are untested.
  - `omnivoice`: tags such as `[laughter]` and `[sigh]`, untested.
  Other engines read a tag aloud as words, so leave tags out for them.

### Personality

A profile with `has_personality: true` can rewrite text in character through a small language model. `speak` sends the text as written, so the wording stays exact.

## Sharing the GPU

`speak` and `transcribe` refuse while the server's large language model is loaded, and the error says so. Do not retry in a loop. Tell the user that the voice engines are waiting for the GPU and that the language model must be unloaded first; only the server's owner can do that.

When you finish a batch of speech work, call `unload_voice_engines` so the GPU is free for the language model. `gpu_status` shows which voice engines and language models hold memory now.

One generation runs at a time. Make `speak` calls one after another, never in parallel.

## Errors

- "no voice named ...": call `list_voices` and use an exact name.
- The text is over 5,000 characters: split it.
- A `transcribe` error about a model download: Whisper is fetching its model; try once more after a minute.
- The audio link returns 403: it has expired or was altered. Call `speak` again.
