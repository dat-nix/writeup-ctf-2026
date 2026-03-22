# CTF Writeup: Choppy

**Category:** Audio Forensics  
**Flag:** `gigem{slic3d&d1ced}`

---

## Challenge Description

> *"I was trying to listen to my friend's new song, but it seems to get all scrambled up after the first second. Can you unscramble it for me?"*

We're given `choppy.wav` — 32 seconds of audio that sounds completely unintelligible after the first second.

---

## Step 1: Spectrogram Analysis

Opening `choppy.wav` in Audacity and viewing the spectrogram immediately reveals:

- **A diagonal line rising steadily** from ~2700Hz to ~6500Hz across the full 32 seconds — this is a **chirp/sweep tone** embedded into every chunk
- Vocal harmonics are still present but buried under the sweep tone and shuffled out of order

This is the key insight: **the sweep tone encodes each chunk's original position as a frequency**.

### File properties

```python
import scipy.io.wavfile as wav
sr, data = wav.read('choppy.wav')
# Sample rate: 44100 Hz | Duration: 32s | int32
# Total: 1,411,200 samples = 128 chunks × 11,025 samples (0.25s/chunk)
```

The file consists of **128 chunks × 0.25 seconds**. The first ~1 second is unmodified. The remaining 127 chunks are randomly shuffled.

---

## Step 2: Unscrambling via Sweep Tone

### How it works

Each chunk contains a sine wave whose frequency is proportional to its **original position** in the audio:

```
Original chunk #0:   dominant freq ≈ 2804 Hz
Original chunk #1:   dominant freq ≈ 2820 Hz
Original chunk #2:   dominant freq ≈ 2840 Hz
...
Original chunk #127: dominant freq ≈ 6244 Hz
```

Sorting chunks by ascending frequency restores the original order.

### Unscramble script

```python
import scipy.io.wavfile as wav
import numpy as np

sr, data = wav.read('choppy.wav')
chunk_size = sr // 4  # 11025 samples = 0.25s

def dominant_freq(chunk, sr):
    fft = np.abs(np.fft.rfft(chunk))
    freqs = np.fft.rfftfreq(len(chunk), 1/sr)
    # Restrict to 2000–7000Hz to avoid vocal harmonics
    mask = (freqs >= 2000) & (freqs <= 7000)
    return freqs[np.argmax(fft * mask)]

# Get dominant frequency of each chunk
freqs = []
for i in range(128):
    chunk = data[i*chunk_size:(i+1)*chunk_size].astype(np.float64)
    freqs.append(dominant_freq(chunk, sr))

# Sort by frequency → restores original order
order = np.argsort(freqs)
pieces = [data[i*chunk_size:(i+1)*chunk_size] for i in order]
result = np.concatenate(pieces)
wav.write('choppy_unscrambled.wav', sr, result.astype(np.int32))
```

**Verification:** Plotting dominant frequency by chunk index after sorting gives a perfectly monotonic ascending line, with `non-monotonic steps = 0`. Unscrambling successful.

---

## Step 3: Vocal Isolation with Demucs

Even after unscrambling, the sweep tone still overlaps with the vocal frequency range, causing Whisper to misread the flag. A manual band-pass filter (300–3400Hz) is not sufficient since the sweep tone sits right on top of the vocal range. We use **Demucs** by Meta for deep learning-based stem separation:

```bash
pip install demucs
demucs --two-stems=vocals choppy_unscrambled.wav
# Output: separated/htdemucs/choppy_unscrambled/vocals.wav
```

> ⚠️ **Adobe Podcast Enhance doesn't work here:** It alters the audio content during enhancement, causing Whisper to produce a completely wrong flag. For CTF challenges, content must be preserved exactly — Demucs only separates stems without modifying them.

---

## Step 4: Speech-to-Text with Whisper

```python
import whisper

model = whisper.load_model("large")  # base/medium misread individual letters
result = model.transcribe("vocals.wav", language="en")
print(result["text"].strip())
```

Output:
```
B I G E M L S curly brackets S L I C 3 D ampers M D 1 C D right curly brackets
```

The flag is spoken **letter by letter**. The `large` model is required — `base` and `medium` frequently misidentify characters with low-quality audio.

---

## Step 5: Decoding the Flag

| Whisper output | Decoded | Notes |
|---|---|---|
| `B I G E M` | `gigem` | Flag prefix |
| `curly brackets` / `right curly bracket` | `{` and `}` | |
| `S L I C 3 D` | `slic3d` | Leet speak: "sliced" |
| `ampers` | `&` | "ampersand" = "and" |
| `D 1 C E D` | `d1ced` | Leet speak: "diced" |

**"Sliced & Diced"** — a cooking phrase that perfectly matches the challenge name **"Choppy"**.

```
gigem{slic3d&d1ced}
```

---

## Key Takeaways

- **Check the spectrogram first:** The sweep tone is immediately visible as a rising diagonal line — spotting it early saves a lot of time.
- **Sweep tone = chunk order:** The scrambler embeds a sine wave into each chunk whose frequency encodes its original position. Sorting by frequency is all that's needed — no cross-correlation or ML required.
- **Demucs over manual filtering:** When the sweep tone overlaps with the vocal range, only deep learning separation can cleanly isolate the voice.
- **Adobe Enhance alters content:** Unsuitable for CTF audio — use Demucs to preserve the original content.
- **Whisper `large` is necessary:** Letter-by-letter spelling with degraded audio quality demands the largest model.
- **Cross-reference + leet speak context:** Run Whisper multiple times and use word meaning to resolve ambiguous characters.
