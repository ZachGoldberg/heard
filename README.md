# heard

A local, real-time transcript of everything you hear and say on your Linux desktop, so you can search for what
someone said three minutes ago, whichever app it came from.

```
$ heard --last 3m
Mon 10:08:26  them: I think the staging rollout is blocked on the IAM change.
Mon 10:08:36    me: Okay, I'll ping the platform team after this.
$ heard "staging rollout"
$ heard -f          # follow live, with an in-progress line for whoever is talking right now
```

- **Any app:** Zoom, Slack huddles, Google Meet, a browser video. It records the system's default output, not
  a particular app.
- **Both sides:** "them" is everything played to your default output; "me" is your default microphone.
- **Local only:** audio never leaves the machine and is never written to disk. Speech is transcribed by
  [faster-whisper](https://github.com/SYSTRAN/faster-whisper) on your GPU (or CPU) and only the text is stored
  in a SQLite full-text index under `~/.local/share/heard/`. Lines older than 7 days are deleted.
- **Read-only capture:** it uses `parec` on the output's monitor source and on the microphone. It never loads
  modules or moves streams, so it is safe to start in the middle of a call.

## Requirements

- Linux with PulseAudio, or PipeWire with `pipewire-pulse` (`parec`, `pactl`)
- [uv](https://docs.astral.sh/uv/). `heard-daemon` is a uv script and installs its own dependencies on the first
  run (faster-whisper, numpy, and NVIDIA's pip-packaged cuBLAS/cuDNN)
- Optional: an NVIDIA GPU. The default model is `large-v3-turbo` on CUDA. Without a usable GPU it falls back to
  `small.en` on the CPU automatically.
- systemd user session (optional, for running it in the background)

## Install

```sh
git clone https://github.com/ZachGoldberg/heard
install -m 755 heard/heard heard/heard-daemon ~/.local/bin/
install -m 644 heard/systemd/heard.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now heard.service
journalctl --user -u heard -f      # first start downloads the model (~1.5 GB for large-v3-turbo)
```

## Usage

| Command | |
|---|---|
| `heard --last 3m` | everything from the last 3 minutes (`90s`, `2h`, `1d` work too) |
| `heard budget` | search ([SQLite FTS5 syntax](https://www.sqlite.org/fts5.html#full_text_query_syntax): `"exact phrase"`, `deploy OR release`) |
| `heard --last 10m budget` | search within a time window |
| `heard -f` | follow new lines live; in a terminal, a dimmed line shows the sentence in progress |
| `heard pause` / `heard resume` / `heard status` | stop and restart capture (e.g. for calls you shouldn't record) |

## How it works

`heard-daemon` reads 16 kHz mono audio from two `parec` streams and cuts each into utterances with a simple
energy gate (a 0.5 s pause ends one; long monologues are cut every 10 s). Finished utterances are transcribed
with voice-activity filtering and stored. While someone is still talking, the utterance so far is re-transcribed
about once a second into `live.json`, so `heard -f` can show live captions. Finished lines always take priority.

It holds a systemd-logind *delay* inhibitor and exits just before suspend. systemd restarts it after resume with a
fresh CUDA context. Keeping a CUDA context across suspend/resume once faulted the NVIDIA driver into "reboot
required".

Environment overrides: `HEARD_MODEL`, `HEARD_CPU_MODEL`, `HEARD_CPU_THREADS`, `HEARD_RETENTION_DAYS`,
`HEARD_THEM_SOURCE` (a fixed source instead of following the default output), `HEARD_ME_SOURCE`.

## Consent

This transcribes other people's speech. Many places, including California, require everyone's consent to record
a confidential conversation, and a transcript likely counts. Tell people, follow your employer's policy, and use
`heard pause` when in doubt.

## License

MIT
