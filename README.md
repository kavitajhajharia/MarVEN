# MarVEN: Marine Vessel and Environmental Noise for Speech

A controlled, reproducible benchmark of clean human speech mixed with real marine surface
noise at calibrated signal to noise ratios, for training and evaluating speech enhancement and
noise robust speech systems that must operate on vessels, at the coast, and at the sea surface.

DOI: https://doi.org/10.5281/zenodo.20714212
---

## 1. Overview

Public corpora fall into two groups that do not overlap. Underwater acoustic corpora are built
for vessel and target classification and contain no human speech. Speech enhancement benchmarks
pair clean speech only with terrestrial, indoor, or urban noise, and contain no marine noise. No
public benchmark pairs clean human speech with marine noise under controlled SNR with splits
that keep speakers apart. MarVEN fills that gap.

Clean speech is drawn from LibriSpeech `train-clean-100`. Noise is drawn from the Discovery of
Sound in the Sea (DOSITS) archive, from recordings contributed by named research groups. Speech
and noise are combined by additive mixing at calibrated SNR levels, producing aligned noisy and
clean pairs suitable for supervised enhancement.

The task is framed as speech robustness in and near the sea, meaning the surface and near
surface interference a microphone actually meets on a vessel or at the coast. It does not model
speech transmitted through water, and no propagation, multipath, or absorption channel is
applied.

### Why these five conditions

The five noises fall into two families that dominate the sound field at and near the sea
surface:

- **Natural weather:** rainfall, lightning, and hurricane.
- **Human marine activity:** a large commercial ship, and a bubble curtain used to reduce noise
  during marine construction.

A speech front end running on a vessel, at a dock, or at the coast is exposed mainly to these
two families: weather overhead, and human activity in the surrounding water. Concentrating on a
coherent, well characterised set of conditions, rather than a long tail of rare sounds, keeps
every condition individually meaningful and keeps the benchmark focused. These five are also the
conditions whose source recordings could be cleared for redistribution (see Licensing).

---

## 2. Dataset at a glance

| Property | Value |
|---|---|
| Clean speech source | LibriSpeech `train-clean-100` |
| Speakers | 100 |
| Clean utterances | 11,377 |
| Noisy files | 22,754 |
| Aligned clean target files | 22,754 |
| Total audio files | 45,508 |
| Noisy audio duration | about 79 hours (matched by 79 hours of clean targets) |
| Noise source | DOSITS, 5 conditions |
| Conditions per utterance | 2 sampled (noise, SNR) combinations |
| SNR levels (dB) | -15, -10, -5, 0, 5, 10, 15, 20 |
| Sample rate | 16 kHz, mono |
| Mixing | Additive, calibrated to a target SNR using an active speech level |
| Splits | By speaker, frozen (about 80/10/10) |
| SNR fidelity | mean \|measured minus target\| = 0.000 dB |
| Speaker leakage | none (verified) |
| Dataset licence | CC BY-NC 4.0 (see Licensing) |

### Per split file counts
| Split | Files |
|---|---|
| train | 18,176 |
| val | 2,298 |
| test | 2,280 |

### Per noise file counts
| Noise | Family | Files |
|---|---|---|
| rainfall | natural weather | 4,587 |
| lightning | natural weather | 4,583 |
| hurricane | natural weather | 4,519 |
| bubble_curtain | human marine activity | 4,570 |
| large_commercial_ship | human marine activity | 4,495 |

### Per SNR file counts
| SNR (dB) | Files |
|---|---|
| -15 | 2,792 |
| -10 | 2,805 |
| -5 | 2,790 |
| 0 | 2,846 |
| 5 | 2,859 |
| 10 | 2,844 |
| 15 | 2,932 |
| 20 | 2,886 |

---

## 3. Intended use and scope

**Intended uses.** Training and benchmarking single channel speech enhancement and denoising
models. Evaluating noise robust ASR or speaker front ends. Robustness analysis by condition and
by SNR for systems meant to run in marine, coastal, shipboard, or surface diver settings.

**Out of scope.** MarVEN does not model speech transmitted through an underwater channel, since
no propagation, multipath, or absorption is applied. It is single channel, so it does not
support array methods such as beamforming.

---

## 4. Directory structure

```
MarVEN/
├── noisy/<speaker>/<chapter>/<spk-chap-utt>_<noise>_snr<XX>.wav   # model input
├── clean/<speaker>/<chapter>/<spk-chap-utt>_<noise>_snr<XX>.wav   # aligned target
└── manifest.csv                                                   # full metadata index
```

- `noisy/` and `clean/` are parallel trees: every noisy file has a clean target of the same
  name at the same relative path, aligned in time and consistent in amplitude.
- Filenames encode speaker, chapter, utterance, noise, and target SNR.

---

## 5. Manifest schema (`manifest.csv`)

One row per noisy file. The manifest is the canonical index; load and filter from it rather than
walking folders.

| Column | Description |
|---|---|
| `noisy_path` | Path to the noisy file, relative to the dataset root |
| `clean_path` | Path to the aligned clean target, relative to the dataset root |
| `clean_source` | Absolute path of the originating LibriSpeech `.flac` |
| `speaker_id` | LibriSpeech speaker ID |
| `chapter_id` | LibriSpeech chapter ID |
| `utterance_id` | Utterance index within the chapter |
| `noise_type` | One of the five noise conditions |
| `target_snr_db` | Requested SNR in dB |
| `measured_snr_db` | Achieved SNR in dB (active speech level) |
| `noise_offset_sec` | Random start offset into the noise recording |
| `mix_scale` | Clip protection scale applied to both mixture and target |
| `sample_rate` | 16000 |
| `duration_sec` | Utterance duration in seconds |
| `split` | `train`, `val`, or `test` (assigned by speaker) |

---

## 6. Construction method (summary)

1. Resample all speech and noise to 16 kHz mono.
2. Sample conditions: each clean utterance is assigned 2 random (noise, SNR) combinations,
   seeded for reproducibility.
3. Random noise offset: the noise segment is taken from a random start position, and looped if
   the recording is shorter than the utterance, to avoid uniform alignment artifacts.
4. Calibrated additive mixing: the noise is scaled so that the ratio of active speech RMS to
   noise RMS equals the target SNR, then the mixture is speech plus scaled noise.
5. Clip protection: if the mixture peak exceeds the ceiling, both the mixture and the clean
   target are scaled by the same factor (`mix_scale`), which preserves the SNR and keeps the
   pair aligned.
6. Splits are assigned by speaker and frozen, so the same speaker never appears in two splits.

No denoising, beamforming, spectral subtraction, or speech filtering is applied during
construction. The noisy files are the unprocessed model inputs and the clean files are the
unprocessed targets.

---

## 7. Loading example

```python
import pandas as pd, soundfile as sf, os

ROOT = "/path/to/MarVEN"
df = pd.read_csv(os.path.join(ROOT, "manifest.csv"))

# example: training pairs at SNR 0 dB or below only
subset = df[(df.split == "train") & (df.target_snr_db <= 0)]

for _, r in subset.iterrows():
    noisy, sr = sf.read(os.path.join(ROOT, r.noisy_path))
    clean, _  = sf.read(os.path.join(ROOT, r.clean_path))
    # noisy is the model input, clean is the target
```

To use fewer speakers, filter `df.speaker_id`; no regeneration is needed.

---

## 8. Splits

Splits are assigned by speaker (about 80/10/10) and frozen, so no speaker appears in more than
one split. This prevents speaker leakage, which would otherwise inflate reported performance.
Verified: no speaker appears in multiple splits.

---

## 9. Licensing and attribution

**Compiled dataset licence: CC BY-NC 4.0.** Four of the five source recordings are distributed
by their holders under Creative Commons noncommercial licences, so the compiled dataset is
released for noncommercial use with attribution. Commercial use of the audio is not permitted
under this release.

**Clean speech.** LibriSpeech is distributed under CC BY 4.0 and is derived from public domain
LibriVox audiobooks. The clean targets are redistributed here with attribution to Panayotov et
al., 2015. CC BY content may be included inside a noncommercial compilation.

**Noise recordings.** Each recording was obtained through the DOSITS audio archive and is
credited to its original holder below. Please retain these credits in any use of the dataset.

| Noise | Recording holder | Basis for inclusion | Credit line |
|---|---|---|---|
| rainfall | Sonatech, Inc. | CC BY-NC | Rainfall recording courtesy of Sonatech, Inc., via DOSITS (CC BY-NC) |
| lightning | Henry Bass, Roy Arnold, and Anthony Atchley, National Center for Physical Acoustics | CC BY-NC | Lightning recording courtesy of H. Bass, R. Arnold, and A. Atchley, NCPA, via DOSITS (CC BY-NC) |
| hurricane | David Mann, Loggerhead Instruments | Included with the written permission of the recordist | Hurricane Irma 2017 recording courtesy of David Mann, Loggerhead Instruments, used with permission |
| bubble_curtain | JASCO Applied Sciences | CC BY-NC | Bubble curtain recording courtesy of JASCO Applied Sciences, via DOSITS (CC BY-NC) |
| large_commercial_ship | Thomas R. Kieckhefer | CC BY-NC | Large commercial ship recording courtesy of Thomas R. Kieckhefer, via DOSITS (CC BY-NC) |

We acknowledge the Discovery of Sound in the Sea (DOSITS) archive, University of Rhode Island
Graduate School of Oceanography, as the archive through which the noise recordings were located,
and we thank the holders above for permitting noncommercial research use.

---

## 10. Known limitations

- **Synthetic additive mixing.** The dataset models speech together with marine background noise,
  not speech recorded through a marine channel. A gap between synthetic and real conditions
  applies, as it does for all synthetic enhancement corpora.
- **Single channel.** No spatial cues, so array methods such as beamforming are not supported. A
  multichannel version is future work.
- **Looping of short clips.** Some source recordings are shorter than some utterances and are
  looped to fill them. For the impulsive lightning recording this can introduce audible
  periodicity; the continuous recordings loop without an obvious seam.
- **Bandlimited speech.** LibriSpeech is 16 kHz read audiobook speech, so spectral content is
  limited accordingly.

---

## 11. Citation (fill in on release)

```bibtex
@misc{marven2026,
  title        = {MarVEN: Marine Vessel and Environmental Noise for Speech},
  author       = {<authors>},
  year         = {2026},
  howpublished = {<Zenodo or Hugging Face>},
  doi          = {<DOI>}
}
```

Please also cite the source corpora: LibriSpeech (Panayotov et al., 2015) and DOSITS, and retain
the per source credits in Section 9.

---

## 12. Changelog

- **v1.0**: initial 100 speaker build, 8 SNR levels (-15 to 20 dB), 5 noise conditions,
  22,754 noisy and clean pairs, splits with no shared speakers, SNR fidelity 0.000 dB, released
  under CC BY-NC 4.0.
