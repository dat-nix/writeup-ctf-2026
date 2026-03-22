# Writeup: numbers — CTF Challenge

**Author:** beds  
**Flag:** `gigem{fft_is_50_0p}`  
**Category:** Misc / Signal Processing  

---

## Challenge Description

> Some stupid numbers or something. Not really sure what they're for, maybe you can figure it out?  
> NOTE: The flag is fully lowercase!  
> HINT: the flag contains the letter 'i'

Attachment: `numbers.txt`

---

## Analysis

Opening the file, the very first line gives everything away:

```
# STFT shape: complex64 (129, 1380)
 (0.000000000000000000e+00+0.000000000000000000e+00j)
 (0.000000000000000000e+00+0.000000000000000000e+00j)
 (-1.484999775886535645e+00+0.000000000000000000e+00j)
 ...
```

The file contains **178,020 complex numbers** — exactly `129 × 1380` — which is **STFT (Short-Time Fourier Transform)** data from an audio file. In other words, someone took an audio clip, ran `stft.spectrogram()` on it, and dumped the result into this text file.

From the shape `(129, 1380)`:
- `129` frequency bins → `framelength = (129 - 1) × 2 = 256`
- `1380` time frames

---

## Solution

### Step 1: Parse the data

The numbers are formatted as `(real+imagj)` with spaces inside the parentheses. These spaces need to be removed before Python can parse them:

```python
val = complex(line.replace(" ", ""))
```

Important note: **the imaginary parts are not all zero** — 165,481 out of 178,020 values have non-zero imaginary components. Ignoring the imaginary part produces a severely distorted ISTFT output.

### Step 2: Identify the original library

The challenge was encoded using the **`stft`** library by Nils Werner — not `scipy.signal.istft`. This matters because the two libraries use different conventions:

- Default window: `scipy.signal.cosine`
- Default overlap: `2` (hop = framelength // 2 = 128) — **not 4**
- `halved=True`: only positive frequencies are stored (129 bins)
- `centered=True`: the signal is center-padded before framing

Using the wrong library or wrong parameters produces audio that is unintelligible.

### Step 3: Reconstruct the audio

The `stft` library is incompatible with Python 3.14+ (it uses `numpy.lib.pad` and `scipy.real`, both of which have been removed). The fix is to reimplement `ispectrogram` from scratch:

```python
import numpy as np
import scipy.fftpack
import scipy.io.wavfile as wav

# Parse
data = []
with open("numbers.txt") as f:
    for line in f:
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        data.append(complex(line.replace(" ", "")))

arr = np.array(data, dtype=np.complex64)
stft_data = arr.reshape(129, 1380)

# Parameters (stft library defaults)
framelength = 256
overlap     = 2
hop         = framelength // overlap  # 128

# Cosine window: sin(pi*(n+0.5)/N)
window = np.sin(np.pi * (np.arange(framelength) + 0.5) / framelength).astype(np.float32)

n_bins, n_frames = stft_data.shape

# Overlap-add reconstruction
out_len = (n_frames - 1) * hop + framelength
out     = np.zeros(out_len, dtype=np.float64)
win_sq  = np.zeros(out_len, dtype=np.float64)

for j in range(n_frames):
    frame_spec = stft_data[:, j]
    # Mirror positive freqs to full spectrum (halved=True)
    full_spec = np.concatenate([frame_spec, np.conj(frame_spec[-2:0:-1])])
    # IFFT
    frame_time = np.real(scipy.fftpack.ifft(full_spec))
    frame_win  = frame_time * window
    start = j * hop
    out[start:start + framelength]    += frame_win
    win_sq[start:start + framelength] += window ** 2

# OLA normalization
nz = win_sq > 1e-12
out[nz] /= win_sq[nz]

# Remove center padding (centered=True)
pad = framelength // 2
out = out[pad:-pad]

# Normalize & save
out /= np.max(np.abs(out))
wav.write("flag.wav", 22050, (out * 32767).astype(np.int16))
```

### Step 4: Speech-to-text with Whisper

After trying several sample rates, `flag_overlap2_sr22050.wav` produced the clearest result:

```bash
pip install openai-whisper
whisper flag_overlap2_sr22050.wav --language English
```

Output:

```
GIGEM left curly bracket FFT underscore I S underscore 5 0 underscore 0 P right curly bracket
```

Assembling the flag in lowercase:

```
gigem{fft_is_50_0p}
```

---

## Flag

```
gigem{fft_is_50_0p}
```

---

## Key Takeaways

- Always read comments in the file — the line `# STFT shape: complex64 (129, 1380)` is the key to everything.
- When reconstructing a signal, **never discard the imaginary part** of complex numbers.
- You must use the **exact same library** the challenge author used to encode — here, Nils Werner's `stft` library with `overlap=2` (default), not `scipy.signal.istft`.
- For audio CTF challenges, **Whisper** is an extremely effective speech-to-text tool, especially when the audio has mild distortion.
