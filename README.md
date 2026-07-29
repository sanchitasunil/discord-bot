# discord-voice-bot

A live voice AI you can join on a Discord Stage and actually talk to. It
listens in the voice channel, replies out loud in real time, switches its
own voice and language mid-conversation, tells apart who is speaking, and
you can interrupt it by talking over it.

It is built on [Hermes](https://github.com/NousResearch/hermes-agent) for
the agent runtime, an LLM for the conversation, a streaming text-to-speech
provider for the voice (this build uses Murf Falcon, swappable), and
Whisper for speech-to-text. This repo is the reproducible path: a pinned
Hermes version, the config and persona files to copy, and a set of patches
you apply on top, safest first.

<!-- TODO: demo GIF here: a Stage session where someone asks a question, the
bot answers, the voice is switched live, and someone interrupts it. -->

## Pin the Hermes version first

Everything here was built and tested against one specific Hermes commit.
The patches are diffed against it. Start by checking it out:

```bash
git clone https://github.com/NousResearch/hermes-agent ~/.hermes/hermes-agent
cd ~/.hermes/hermes-agent
git checkout 0f102fa
```

Note: this commit is not tagged upstream, so the pin is a commit SHA, not a
release tag. Newer Hermes may already contain some of the general bug fixes
below (the ones marked `[upstream-pr]`) - check before applying those.

Windows: do all of this inside WSL, where `~/.hermes/` resolves normally.
Paths and commands are otherwise identical, so there is no separate Windows
track.

## What is in this repo

```
discord-voice-bot/
  README.md
  config/          # files you COPY into your Hermes home
    config.yaml
    SOUL.md
    .env.example
  patches/         # diffs you APPLY against the pinned Hermes
```

## The changes at a glance

| Change | Type | Status | You do |
|---|---|---|---|
| Stage channels: don't try to auto-thread (bot can't) | `[upstream-pr]` | not yet filed | apply `001` |
| Multi-speaker SSRC attribution fix | `[upstream-pr]` | not yet filed | apply `002` |
| Whisper energy gate (stop hallucinating on silence) | `[upstream-pr]` | not yet filed | apply `003` |
| `reasoning.effort` 400 on non-reasoning OpenAI models | `[upstream-pr]` | not yet filed | apply `004` |
| Slash-command sync: retry after a failed sync | `[upstream-pr]` | not yet filed | apply `005` |
| Voice bot features (streaming, voice + language switching, barge-in, attribution, recap) | `[patch]` | fork-level | apply `006` |
| `config.yaml` | `[config]` | yours to copy | copy + edit |
| `SOUL.md` (persona) | `[config]` | yours to copy | copy + customize |
| `.env.example` | `[config]` | yours to copy | copy to `.env`, fill in |

`[upstream-pr]` = a general Hermes bug fix that belongs upstream and may
already be merged in a newer Hermes. `[patch]` = fork-level, specific to
building this kind of voice bot. `[config]` = a file you copy, not a diff.

## 1. Install Hermes and the TTS plugin

Follow the Hermes install for the pinned checkout above. Then install the
TTS provider plugin from PyPI:

```bash
pip install hermes-plugin-murf-tts
```

- PyPI: https://pypi.org/project/hermes-plugin-murf-tts/
- Source: https://github.com/sanchitasunil/hermes-murf-plugin

The plugin implements Hermes's `TTSProvider` interface, so the bot is not
married to Murf. Swap in any provider that implements the same interface and
the rest of this repo still applies. What Murf specifically gives you here
is sub-100ms streaming and one voice that speaks many languages, which the
language-switching feature leans on.

## 2. Copy the config

Copy the three files from `config/` into your Hermes home:

```bash
cp config/config.yaml ~/.hermes/config.yaml
cp config/SOUL.md     ~/.hermes/SOUL.md
cp config/.env.example ~/.hermes/.env      # then edit ~/.hermes/.env and fill in real keys
```

- `config.yaml` - the default model is `gpt-4o`, a non-reasoning model
  chosen for low, predictable latency (see patch `004` for why reasoning
  models are a bad fit for live voice). No secrets live here; keys go in
  `.env`.
- `SOUL.md` - the persona. It ships generic on purpose. Rename the bot,
  give it a personality, but keep the "How you speak", voice-switching, and
  language-switching sections roughly as-is; those are what make it work as
  a voice bot.
- `.env` - every key the bot needs, with a one-line comment each. Never
  commit the filled-in file.

## 3. Get a working bot before you touch the patches

At this point, join a voice channel and start the gateway:

```bash
cd ~/.hermes/hermes-agent && hermes gateway
```

Invite the bot to your server, `/voice join`, and talk to it. You now have
a working conversational voice bot. Everything below is incremental
improvement, and each patch is independently valuable and skippable. You can
stop after patch `001` with a working Stage bot, or after `003` with clean
transcription, and come back for the rest later.

## 4. Apply the patches, safest first

Apply from inside the pinned Hermes checkout:

```bash
cd ~/.hermes/hermes-agent
git apply /path/to/discord-voice-bot/patches/001-stage-thread-fix.patch
```

Check first with `git apply --check <patch>` if you want a dry run. Restart
the gateway after applying to load the changes.

### 001 - Stage channels: skip auto-threading `[upstream-pr]`

What it fixes: the bot joining a Stage channel and failing on every message
because Hermes tries to auto-create a thread, which Discord forbids on Stage
channels (400 / error 50024).

Why it broke: the auto-thread-on-mention behavior did not special-case Stage
channels. It is a permanent Discord limitation, not a transient error, so
the right fix is to skip thread creation there rather than retry.

```bash
git apply patches/001-stage-thread-fix.patch
```

May already be merged upstream; check before applying.

### 002 - Multi-speaker SSRC attribution `[upstream-pr]`

What it fixes: with two or more people in the voice channel, the bot could
only hear one of them. Audio from unmapped speakers was silently dropped.

Why it broke: Discord maps an audio stream to a user via a SPEAKING event
that only fires on a silence-to-speech transition. If that event never
arrives (someone was already talking when the bot joined), a fallback tries
to guess the speaker. The fallback only worked when exactly one person was
in the channel, so it broke the instant a second person joined. The fix
narrows the guess to the one remaining unmapped speaker, which works with
any number of people.

```bash
git apply patches/002-multispeaker-ssrc.patch
```

May already be merged upstream; check before applying.

### 003 - Whisper energy gate `[upstream-pr]`

What it fixes: the bot "hearing" speech during silence and replying to
sentences nobody said.

Why it broke: Discord relays background noise and room echo as real audio
packets. Feeding quiet, non-speech audio to Whisper does not produce an
error; it produces confident, hallucinated transcripts ("Gracias.", "Tell
me something about births."). Packet presence alone cannot tell speech from
noise. The fix adds a peak-RMS loudness gate before any audio reaches
Whisper, with zero new dependencies (stdlib only).

```bash
git apply patches/003-whisper-energy-gate.patch
```

May already be merged upstream; check before applying.

### 004 - `reasoning.effort` on non-reasoning OpenAI models `[upstream-pr]`

What it fixes: an HTTP 400 "Unsupported parameter: 'reasoning.effort'" on
every request when using a non-reasoning chat model (gpt-4o, gpt-4o-mini,
the gpt-5.x-chat variants) through the OpenAI Responses transport.

Why it broke: the transport attached `reasoning.effort` to every OpenAI
model with no capability check. Non-reasoning models reject it. This
matters for voice because reasoning models add several seconds of hidden
"thinking" latency per turn, which is dead air in a live conversation, so
you want a non-reasoning model, and those were exactly the ones that 400ed.
The fix adds a check for the model families that reject the parameter.

```bash
git apply patches/004-reasoning-effort-fix.patch
```

Two files: `agent/model_metadata.py` and `agent/transports/codex.py`. May
already be merged upstream; check before applying.

### 005 - Slash-command sync retry `[upstream-pr]`

What it fixes: slash commands silently never re-syncing after one failed
sync, so `/voice` and friends never appear.

Why it broke: the sync-skip logic treated "a sync was attempted" as "a sync
succeeded". After a failed attempt it recorded state that made it skip
forever. The fix records a separate "succeeded" fingerprint so a failed sync
is retried.

```bash
git apply patches/005-command-sync-retry.patch
```

May already be merged upstream; check before applying.

### From here we are editing Hermes internals

The patches above are self-contained bug fixes. The next one is the
fork-level work: the actual voice-bot features. It is a larger, coupled
change that touches the Discord adapter and the gateway conversation loop.

### 006 - Voice bot features `[patch]`

What it adds, in one patch:

- Sentence-by-sentence streaming TTS over a persistent WebSocket, so the bot
  starts speaking as the reply is generated instead of after it finishes.
- Conversational voice switching and previewing via `[[SWITCH_VOICE]]` and
  `[[SAMPLE_VOICE]]` markers the model emits in its reply, resolved and
  stripped before anyone sees them.
- Live language switching via a `[[SPEAK_LANGUAGE]]` marker, using one
  multilingual voice with no reconnect.
- Speaker attribution: real display names in the shared conversation history
  so the bot follows who asked what and can address people by name.
- End-of-session recap: an optional summary posted to a text channel when
  the voice session ends.
- Barge-in: keep listening during playback and halt fast when someone talks
  over the bot, then keep the interrupted context.

```bash
git apply patches/006-fork-voice-features.patch
```

Honest note on why this is one patch and not four. These features were
built incrementally in the same code paths (the `LiveSpeechSession`, the
gateway's per-turn loop), so in the working tree they are interleaved at the
line level and cannot be cleanly separated into independent, individually
applying diffs after the fact. They are documented as separate features
above, and the lesson is real: if you want per-feature patches, build each
on its own branch from the start. One of these, barge-in, is architecturally
adapter-only (no agent-core change) and would make a strong standalone
upstream PR once extracted from a clean branch: it makes the listener-pause
configurable and adds interruption handling.

## Verify

After applying and restarting, the fastest end-to-end check:

1. `/voice join`, say hi. It should reply out loud within a second or two.
2. "Switch to <a voice name>." It should confirm in the new voice.
3. Speak to it in a language its voice supports. It should answer in that
   language, then return to the default on your next English turn.
4. Talk over it mid-reply. It should stop quickly and take your new question.

## Links

- Hermes: https://github.com/NousResearch/hermes-agent
- TTS plugin on PyPI: https://pypi.org/project/hermes-plugin-murf-tts/
- TTS plugin source: https://github.com/sanchitasunil/hermes-murf-plugin
- Upstream PRs for the `[upstream-pr]` fixes: not yet filed. If you file
  them, link them here and update the status column.
