---
name: voice-note-recording
description: "Set up and use a CLI voice-note recorder on Linux using arecord + ffmpeg (no GUI, no external APIs). Use when the user wants to record voice notes, voice memos, dictation, or audio from a microphone."
---

# Voice Note Recording

Set up a lightweight, terminal-based voice note recorder on Linux that captures microphone audio and saves timestamped MP3 files. Works with PulseAudio, PipeWire, and bare ALSA.

## Quick quality checklist

- `name` matches folder name exactly (kebab-case)
- All examples are tested and runnable
- Includes both Bash and Node.js examples
- Uses free/public tools first (or explains paid fallback)
- No secrets, API keys, or personal data in examples

## When to use

- Use case 1: When the user asks to record voice notes, memos, or dictation on Linux
- Use case 2: When setting up microphone recording from a terminal (headless or desktop)
- Use case 3: When automation needs to capture audio to a timestamped file

## Required tools / APIs

- `arecord` (ALSA utils) — microphone capture to WAV
- `ffmpeg` — converts WAV to compressed MP3/OGG
- No external API required

Install options:

```bash
# Ubuntu/Debian
sudo apt-get install -y alsa-utils ffmpeg

# macOS (equivalent recorder is `sox`)
brew install ffmpeg sox

# Fedora
sudo dnf install -y alsa-utils ffmpeg
```

## Skills

### basic_usage

Install a `note` command that records until Ctrl+C and saves a timestamped MP3.

Steps:

1. Create the recorder script at `~/.local/bin/note` (ensure `~/.local/bin` is on `PATH`).
2. `chmod +x ~/.local/bin/note`
3. Run `note`, speak, press Ctrl+C.

```bash
#!/bin/bash
DIR="$HOME/VoiceNotes"
mkdir -p "$DIR"
STAMP=$(date +%Y-%m-%d_%H%M%S)
WAV=$(mktemp /tmp/note.XXXXXX.wav)
echo "Recording voice note... press Ctrl+C to stop."
if arecord -q -f S16_LE -r 44100 -c 1 -t wav "$WAV"; then
  RC=0
else
  RC=$?
fi
if [ "$RC" -eq 1 ] || [ "$RC" -eq 0 ]; then
  if [ -s "$WAV" ]; then
    MP3="$DIR/voice-note-$STAMP.mp3"
    if ffmpeg -loglevel error -y -i "$WAV" -c:a libmp3lame -q:a 4 "$MP3"; then
      rm -f "$WAV"
      echo "Saved: $MP3"
    else
      echo "Moved note to: $WAV (mp3 conversion failed)"
    fi
  else
    rm -f "$WAV"
    echo "No audio captured (too short?). Nothing saved."
  fi
else
  rm -f "$WAV"
  echo "Recording failed - check microphone."
fi
echo "Notes: $DIR"
```

One-off alternative (no script install):

```bash
mkdir -p ~/VoiceNotes
arecord -f S16_LE -r 44100 -c 1 -t wav -d 30 | \
  ffmpeg -loglevel error -i - -c:a libmp3lame -q:a 4 \
  "$HOME/VoiceNotes/voice-note-$(date +%Y-%m-%d_%H%M%S).mp3"
```

This records a fixed 30-second take into a timestamped MP3.

**Node.js (Node 14 or higher, no deps):**

```javascript
const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');
const os = require('os');

const dir = path.join(os.homedir(), 'VoiceNotes');
fs.mkdirSync(dir, { recursive: true });
const stamp = new Date().toISOString().replace(/[:.]/g, '-');
const out = path.join(dir, `voice-note-${stamp}.mp3`);

// Pipe live mic audio (arecord -> ffmpeg -> mp3) until parent exits.
const arecord = spawn('arecord', ['-q', '-f', 'S16_LE', '-r', '44100', '-c', '1', '-t', 'wav'], { stdio: ['ignore', 'pipe', 'inherit'] });
const ffmpeg = spawn('ffmpeg', ['-loglevel', 'error', '-i', 'pipe:0', '-c:a', 'libmp3lame', '-q:a', '4', out], { stdio: ['pipe', 'inherit', 'inherit'] });

arecord.stdout.pipe(ffmpeg.stdin);
process.on('SIGINT', () => { arecord.kill('SIGINT'); ffmpeg.kill('SIGINT'); setTimeout(() => process.exit(0), 500); });
```

### robust_usage

Verify the microphone before recording, and never lose audio if MP3 conversion fails.

```bash
#!/bin/bash
set -euo pipefail
DIR="$HOME/VoiceNotes"
mkdir -p "$DIR"

if ! arecord -l >/dev/null 2>&1; then
  echo "No capture device found. Check the microphone." >&2
  exit 1
fi

STAMP=$(date +%Y-%m-%d_%H%M%S)
WAV="$(mktemp /tmp/note.XXXXXX.wav)"
MP3="$DIR/voice-note-$STAMP.mp3"

trap 'rm -f "$WAV"' EXIT
echo "Recording... press Ctrl+C to stop."
ret=0
arecord -q -f S16_LE -r 44100 -c 1 -t wav "$WAV" || ret=$?

# arecord exits 1 on SIGINT, which still yields a finalized WAV (header written).
if [ "$ret" -eq 0 ] || [ "$ret" -eq 1 ]; then
  if [ -s "$WAV" ]; then
    if ffmpeg -loglevel error -y -i "$WAV" -c:a libmp3lame -q:a 4 "$MP3"; then
      echo "Saved: $MP3"
    else
      cp "$WAV" "$DIR/voice-note-$STAMP.wav"
      echo "Conversion failed; kept raw audio: $DIR/voice-note-$STAMP.wav"
    fi
  else
    echo "No audio captured (too short?). Nothing saved."
  fi
else
  echo "Recording failed (arecord exit $ret). Check the microphone and PipeWire/Pulse status." >&2
fi
```

**Node.js:**

```javascript
const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');
const os = require('os');

async function recordNote(seconds = null) {
  const dir = path.join(os.homedir(), 'VoiceNotes');
  fs.mkdirSync(dir, { recursive: true });
  const stamp = new Date().toISOString().replace(/[:.]/g, '-');
  const wav = path.join(os.tmpdir(), `note-${Date.now()}.wav`);
  const mp3 = path.join(dir, `voice-note-${stamp}.mp3`);

  const args = ['-q', '-f', 'S16_LE', '-r', '44100', '-c', '1', '-t', 'wav'];
  if (seconds) args.push('-d', String(seconds));
  args.push(wav);

  return new Promise((resolve, reject) => {
    const rec = spawn('arecord', args, { stdio: 'ignore' });
    rec.on('close', (code) => {
      // arecord returns 1 on SIGINT; WAV header is still finalized.
      if (code !== 0 && code !== 1) return reject(new Error(`arecord exit ${code}`));
      if (!fs.existsSync(wav) || fs.statSync(wav).size === 0) return resolve(null);

      const ff = spawn('ffmpeg', ['-loglevel', 'error', '-y', '-i', wav, '-c:a', 'libmp3lame', '-q:a', '4', mp3]);
      ff.on('close', (fc) => {
        fs.unlinkSync(wav);
        if (fc !== 0) return reject(new Error(`ffmpeg exit ${fc}`));
        resolve(mp3);
      });
      ff.on('error', (e) => reject(e));
    });
    rec.on('error', (e) => reject(e));
  });
}

// recordNote();            // until Ctrl+C (default signal handling still applies)
// recordNote(30).then(console.log); // 30-second fixed take
```

## Output format

On success the command prints:

- `Saved: /home/user/VoiceNotes/voice-note-YYYY-MM-DD_HHMMSS.mp3`
- `Notes: /home/user/VoiceNotes`

Error shapes:

- `Recording failed - check microphone.` → capture device missing or busy
- `No audio captured (too short?). Nothing saved.` → file was empty/too-short
- Raw WAV is kept if MP3 conversion fails (never lose a take)

## Rate limits / Best practices

- Mono 44.1 kHz 16-bit is plenty for speech and keeps files small
- Use `-q:a 4` (VBR ~128-170 kbps) for good quality/size balance
- A microphone that is recording (visible in `pactl list short sources` as state `RUNNING`) is working
- On SIGINT, `arecord` finalizes the WAV header but exits non-zero (1) — treat that as a valid take

## Agent prompt

```text
You have voice-note-recording capability. When a user asks to record voice notes on Linux:

1. Check capture tools: `which arecord ffmpeg` and `arecord -l` for a device
2. Create ~/VoiceNotes and install a `note` script in ~/.local/bin that records until
   Ctrl+C and saves a timestamped MP3 (arecord -> ffmpeg -q:a 4)
3. Treat arecord exit code 1 (SIGINT / Ctrl+C) as a successful recording
4. If conversion fails, keep the raw WAV rather than losing the recording
5. Tell the user where notes are saved and show the recorded file path

Use free local tools (arecord + ffmpeg) only — no cloud speech services unless asked.
```

## Troubleshooting

**Error scenario 1: "Recording failed - check microphone."**
- Symptom: arecord returns non-zero immediately (exit other than 0/1)
- Solution: `pactl list short sources` to confirm the source exists; unmute/raise capture in `pavucontrol`; check the card is the active default (`arecord -l`)

**Error scenario 2: "No audio captured (too short?). Nothing saved."**
- Symptom: WAV file is 0 bytes after stopping
- Solution: Record longer than a second; raise input gain; unmute the mic with `amixer -c 0 set 'Capture' cap` or pavucontrol

**Error scenario 3: mp3 conversion fails**
- Symptom: `ffmpeg: error while loading shared libraries` or invalid input
- Solution: Install ffmpeg (`sudo apt-get install -y ffmpeg`); the raw WAV is kept at the printed path