# MarVEN: Data Dictionary

**Marine Vessel and Environmental Noise for Speech (MarVEN), v1.0**

This document explains every field in `manifest.csv`, the controlled vocabularies for the
categorical fields, the file naming scheme, and how files are laid out on disk.

Treat `manifest.csv` as the single source of truth for the dataset. Each row describes one noisy
file and the clean target it is paired with. When loading data, read and filter the manifest
rather than walking the directory tree yourself.

---

## 1. Record granularity

Each row in `manifest.csv` represents one noisy mixture file paired with its aligned clean
target. Because each clean utterance is mixed under two randomly sampled conditions (see
Section 3), a given LibriSpeech source utterance appears in two rows, one for each `noise_type`
and `target_snr_db` combination, each with a different `noisy_path`.

---

## 2. Field definitions

| Field | Type | Unit or format | Allowed values or range | Description |
|---|---|---|---|---|
| `noisy_path` | string | POSIX relative path | `noisy/<speaker>/<chapter>/<file>.wav` | Path to the noisy mixture (the model input), relative to the dataset root. |
| `clean_path` | string | POSIX relative path | `clean/<speaker>/<chapter>/<file>.wav` | Path to the aligned clean target for this row. Same relative filename as `noisy_path` but under `clean/`; aligned in time and consistent in amplitude with the mixture. |
| `clean_source` | string | absolute path | | Path of the originating LibriSpeech `.flac` file the clean target was derived from. Provenance only; not needed for loading. |
| `speaker_id` | integer | LibriSpeech ID | LibriSpeech `train-clean-100` speaker IDs | Speaker identifier. Train, validation, and test splits are assigned by this field (see `split`). |
| `chapter_id` | integer | LibriSpeech ID | | LibriSpeech chapter (book section) the utterance belongs to. |
| `utterance_id` | integer | index | at least 0 | Utterance index within the chapter, following LibriSpeech numbering. |
| `noise_type` | string (categorical) | | one of 5 values, see Section 4.1 | The DOSITS noise condition mixed into this file. |
| `target_snr_db` | number | dB | one of 8 values, see Section 4.2 | The requested signal to noise ratio. The noise gain was chosen so that the active speech level SNR reaches this value. |
| `measured_snr_db` | number | dB | close to `target_snr_db` | The SNR recomputed from the actual mixed signals after construction, as a fidelity check. The mean absolute difference from `target_snr_db` is 0.000 dB across the dataset. |
| `noise_offset_sec` | number | seconds | at least 0 | Random start offset into the source noise recording where the noise segment begins. If the recording is shorter than the utterance, it loops from this point. |
| `mix_scale` | number | linear gain | between 0 and 1 | Clip protection factor applied equally to the mixture and the clean target. It is 1.0 when no protection was needed, and less than 1.0 when the mixture peak exceeded 0.99 and both signals were scaled down together. Because it scales the numerator and denominator equally, the SNR is unchanged. |
| `sample_rate` | integer | Hz | 16000 | Sample rate of both the noisy and clean files. Constant across the dataset. |
| `duration_sec` | number | seconds | greater than 0 | Duration of the utterance, and therefore of the mixture. |
| `split` | string (categorical) | | `train`, `val`, or `test` | Which partition this file belongs to. Assigned by speaker, so no speaker ever crosses splits. |

Notes on the derived fields:

- `measured_snr_db` is computed as `20 * log10(RMS_speech / RMS(g * noise))`, where `RMS_speech`
  is the active speech level RMS (voiced frames only) and `g` is the solved noise gain. It is
  there for transparency and auditing, and is not an input to the mixing process.
- Together, `mix_scale` and `measured_snr_db` give everything needed to reconstruct or verify
  any mixture from its source signals.

---

## 3. Sampling design (how the rows are generated)

Each clean utterance is assigned two randomly drawn (`noise_type`, `target_snr_db`) combinations,
with the draw seeded for reproducibility. Choosing two conditions per utterance, rather than the
full crossing of 5 noises by 8 SNR levels, keeps the dataset at a manageable size while still
giving near uniform coverage across all noise conditions and SNR levels. This has been verified:
each condition is within a few percent of the others (see the per noise and per SNR counts in the
README).

The noise segment for each row also starts at a random offset into the source recording, so files
do not all begin at the same point in a given noise.

---

## 4. Controlled vocabularies

### 4.1 `noise_type` (5 conditions)

| Value | Family | Temporal character |
|---|---|---|
| `rainfall` | natural weather | continuous |
| `lightning` | natural weather | impulsive |
| `hurricane` | natural weather | continuous, low frequency |
| `bubble_curtain` | human marine activity | pulsed, quasi periodic |
| `large_commercial_ship` | human marine activity | continuous, broadband |

All five come from the Discovery of Sound in the Sea (DOSITS) archive and are credited to their
holders in the README and datasheet. The temporal character column is informational: continuous
recordings loop without an audible seam, while the impulsive lightning recording can reveal
periodicity when a short clip is looped to match a longer utterance. This is noted as a
limitation.

### 4.2 `target_snr_db` (8 levels)

| Value (dB) | Regime |
|---|---|
| 20 | speech dominant (easy) |
| 15 | speech dominant (easy) |
| 10 | speech clearly above noise |
| 5 | speech above noise |
| 0 | speech and noise at equal power |
| -5 | noise above speech (hard) |
| -10 | noise dominant (stress) |
| -15 | strongly noise dominant (stress) |

The range extends into strongly noise dominant territory (-10 and -15 dB) to provide a genuine
stress regime and a full easy to hard curve. The intended evaluation mode is to report model
performance as a function of `target_snr_db`.

### 4.3 `split` (3 partitions)

| Value | Approximate proportion | Assignment unit |
|---|---|---|
| `train` | about 80 percent | speaker |
| `val` | about 10 percent | speaker |
| `test` | about 10 percent | speaker |

Splits are frozen and speaker based, so the same speaker never appears in two partitions.

---

## 5. File naming convention

```
<speaker>-<chapter>-<utterance>_<noise>_snr<XX>.wav
```

- `<speaker>-<chapter>-<utterance>` follows the LibriSpeech identifier of the source utterance.
- `<noise>` is one of the five `noise_type` values.
- `snr<XX>` encodes the target SNR (for example `snr-5` or `snr10`).
- The noisy file and its clean target share the same filename; one lives under `noisy/`, the
  other under `clean/`.

---

## 6. On disk layout

```
MarVEN/
├── noisy/<speaker>/<chapter>/<spk-chap-utt>_<noise>_snr<XX>.wav   # model input
├── clean/<speaker>/<chapter>/<spk-chap-utt>_<noise>_snr<XX>.wav   # aligned target
└── manifest.csv                                                    # this dictionary describes it
```

`noisy/` and `clean/` are parallel trees: every noisy file has a clean counterpart of the same
name at the same relative path. This mirrors the layout that supervised enhancement trainers
expect, the same shape as VoiceBank+DEMAND style corpora.

---

## 7. Audio format summary

| Property | Value |
|---|---|
| Container | WAV (PCM) |
| Bit depth | 16 bit |
| Sample rate | 16 kHz |
| Channels | 1 (mono) |
| Amplitude range | normalised to the range minus 1 to 1; mixtures peak protected at 0.99 or below |

---

## 8. Minimal load recipe

```python
import pandas as pd, soundfile as sf, os

ROOT = "/path/to/MarVEN"
df = pd.read_csv(os.path.join(ROOT, "manifest.csv"))

# example: training pairs at SNR 0 dB or below only
subset = df[(df.split == "train") & (df.target_snr_db <= 0)]
for _, r in subset.iterrows():
    noisy, sr = sf.read(os.path.join(ROOT, r.noisy_path))   # model input
    clean, _  = sf.read(os.path.join(ROOT, r.clean_path))   # target
```

To train on a subset of speakers, filter on `df.speaker_id`; no regeneration is needed.
