# MarVEN Datasheet

**Marine Vessel and Environmental Noise for Speech (MarVEN), v1.0**

This datasheet follows the structure of *Datasheets for Datasets* (Gebru et al., 2021). It
documents the motivation, composition, construction, intended use, distribution, and maintenance
of the dataset. Together with `manifest.csv` (the per file annotation layer) and
`DATA_DICTIONARY.md` (the field definitions), it forms the documentation package for the release.

---

## Motivation

**For what purpose was the dataset created?**
To provide a controlled, reproducible benchmark of clean human speech paired with real marine
surface noise at calibrated signal to noise ratios, for training and evaluating single channel
speech enhancement and noise robust speech systems meant to run on vessels, at the coast, and at
the sea surface. Existing public corpora do not cover this combination. Underwater acoustic
corpora hold vessel, animal, and ambient recordings for classification but no human speech, while
speech enhancement benchmarks pair clean speech only with terrestrial, indoor, or urban noise.

**What gap does it fill?**
A paired clean speech and marine noise benchmark with explicit SNR control and splits that keep
speakers apart. The mixing methodology is standard and well established; the novelty is the
acoustic domain and the controlled, documented construction.

**Who created the dataset?**
The dataset maintainers (to be named on release). The source corpora were produced by third
parties (LibriSpeech, and the individual recording holders listed under Distribution), and are
credited there.

---

## Composition

**What do the instances represent?**
Each instance is a pair of single channel 16 kHz audio files: a noisy mixture (the model input)
and its aligned clean target. Mixtures are formed by adding scaled marine noise to a clean speech
utterance at a chosen SNR.

**How many instances are there?**
- Speakers: 100 (a subset of LibriSpeech `train-clean-100`).
- Clean source utterances: 11,377.
- Noisy files: 22,754 (each utterance under two sampled conditions).
- Aligned clean target files: 22,754.
- Total audio files: 45,508.
- Noisy audio duration: about 79 hours, matched by about 79 hours of clean targets.
- Noise conditions: 5. SNR levels: 8 (-15 to 20 dB).

**Per split counts.** train 18,176, val 2,298, test 2,280.

**Per noise counts.** rainfall 4,587, lightning 4,583, hurricane 4,519, bubble_curtain 4,570,
large_commercial_ship 4,495. Near uniform across the five conditions, and likewise across the
eight SNR levels; exact counts are in the README and recomputable from the manifest.

**Is each instance labelled?**
Yes. The manifest annotates every file with source speaker, chapter, and utterance identifiers,
`noise_type`, `target_snr_db`, `measured_snr_db`, `noise_offset_sec`, `mix_scale`, `sample_rate`,
`duration_sec`, and `split`. These constitute the per file annotation; see `DATA_DICTIONARY.md`.

**Is any information missing?**
No transcripts are redistributed in v1.0 (they are available from LibriSpeech via the source
identifiers if needed). No per speaker demographic labels beyond what LibriSpeech provides.

**Are there recommended data splits?**
Yes. Frozen train, validation, and test splits assigned by speaker (about 80/10/10), so that no
speaker appears in more than one split. Splitting by file would leak speaker identity across
partitions and inflate measured performance.

**Are there errors, sources of noise, or redundancies?**
By design the dataset is a benchmark of noise: the noisy files are intentionally degraded inputs.
Two construction characteristics are disclosed as limitations: short recordings are looped to
fill longer utterances, which can introduce audible periodicity for the impulsive lightning
recording; and additive mixing models speech together with marine background noise, not speech
transmitted through a marine channel.

**Does the dataset depend on external resources?**
The audio is derived from LibriSpeech and from five DOSITS recordings. The compiled audio is
redistributed here under the terms set out in Distribution (all five recordings are cleared for
noncommercial redistribution).

**Does it contain confidential, sensitive, or offensive content?**
No. LibriSpeech speakers are volunteers reading public domain audiobooks, and users agree not to
attempt to identify individual speakers. The noise recordings are environmental and scientific audio.

---

## Collection process

**How was the data acquired?**
The dataset is synthesised, not recorded. Clean speech was taken from LibriSpeech
`train-clean-100` (read audiobook speech, 16 kHz). Noise was taken from the DOSITS audio archive,
from five recordings spanning two families: natural weather (rainfall, lightning, hurricane) and
human marine activity (a large commercial ship, and a bubble curtain used in marine
construction). The two were combined by calibrated additive mixing.

**Why these five conditions?**
Two reasons, and both are stated plainly. First, the five fall into two families that dominate
the sound field at and near the sea surface, so a speech system on a vessel or at the coast meets
mainly these: weather overhead and human activity in the water. Second, these are the conditions
whose source recordings could be cleared for redistribution. Four are distributed by their
holders under Creative Commons noncommercial licences, and one (hurricane) is included with the
written permission of the recordist. Four further conditions considered during development (waves
on a beach, tidal turbine, scuba, and sonar) were not included, because their recordings are
either all rights reserved or, in the case of scuba, released under a no derivatives Creative
Commons licence that does not permit distributing a mixed derivative.

**Why these sources?**
LibriSpeech provides large, standard, permissively licensed clean targets that reviewers
recognise and that are required for reference based metrics such as PESQ, STOI, and SI-SDR. DOSITS
provides real, scientifically reviewed marine recordings, each with a documented holder.

**Over what timeframe was the data collected?**
The source recordings predate this work. Dataset construction was performed by the maintainers
(date on release). Construction is fully scripted and seeded, so it is reproducible at any time.

**Were people involved, and did they consent?**
LibriSpeech speakers consented to public release of their recordings through the LibriVox and
LibriSpeech process, and the corpus is distributed under CC BY 4.0. No new human subjects data was
collected for MarVEN.

---

## Preprocessing, cleaning, and labelling

**What preprocessing was done?**
1. Resample all speech and noise to 16 kHz mono.
2. Assign each utterance two random (noise, SNR) conditions, seeded.
3. Take noise from a random offset, looping if it is shorter than the utterance.
4. Calibrated additive mixing: measure the active speech level (RMS over voiced frames only,
   silences excluded), solve the noise gain so the speech to noise ratio equals the target SNR,
   then add. With active speech RMS `RMS_s` and noise RMS `RMS_n`, the gain is
   `g = RMS_s / (RMS_n * 10^(SNR/20))` and the mixture is `y = s + g * n`.
5. Clip protection that preserves SNR: if the mixture peak exceeds 0.99, scale both the mixture
   and the clean target by the same factor `mix_scale`; because it cancels in the ratio, the
   target SNR is preserved and the pair stays aligned.
6. Recompute the SNR from the actual signals and log it as `measured_snr_db`.

**Why is the noise scaled rather than used at its raw recorded level?**
Raw recordings have arbitrary, per recording loudness. Adding them unscaled would give every file
a different, uncontrolled SNR, which is not a benchmark. Scaling the noise to a target SNR makes
difficulty an explicit, comparable variable, following the same approach used by MS-SNSD, DNS, and
VoiceBank+DEMAND. An earlier fixed gain version (speech times 1.0 plus noise times 0.6, followed
by peak normalisation) was discarded for exactly this reason: its effective SNR varied per file
with raw clip loudness and was further altered by normalisation.

**What was deliberately not applied?**
- No denoising of any kind is baked into the files; the noisy files stay unprocessed inputs and
  the clean files stay unprocessed targets.
- No beamforming, since the data is single channel and spatial filtering is physically
  inapplicable.
- No spectral subtraction or high pass filtering as curation. These are baselines run on the
  dataset and reported with metrics, not part of building it.
- No artificial underwater lowpass or boost on the speech. An arbitrary cutoff is not a physical
  channel model, and the framing is speech in and near the sea, not speech transmitted through
  water.

**A note on the source recordings.**
The five noise recordings are distributed by DOSITS in a stereo container at 48 kHz, but the two
channels are identical (dual mono), so the recordings carry no spatial information. They were
decoded, downmixed to mono (a lossless step given the identical channels), and resampled to
16 kHz before mixing.

**Is the raw source data retained?**
The originating LibriSpeech file path is logged per row (`clean_source`). The source corpora
remain available from their original distributors.

---

## Uses

**What is the dataset intended for?**
- Training and benchmarking single channel speech enhancement and denoising.
- Evaluating noise robust ASR or speaker front ends under marine noise.
- Robustness analysis by condition and by SNR.

**What should it not be used for?**
- Modelling speech transmitted through an underwater acoustic channel (no propagation, multipath,
  or absorption is applied).
- Array methods such as beamforming (the data is mono).
- Claims of demographic representativeness beyond LibriSpeech's own composition.
- Any commercial use, since the compiled dataset is released under a noncommercial licence.

**What is the recommended reporting protocol?**
Report reference based enhancement metrics (for example wideband PESQ, STOI, and SI-SDR with its
improvement over the input) and a non intrusive MOS estimate (for example DNSMOS), broken down by
SNR level and by noise condition, on the frozen splits. Cross paper comparison is valid only on
the identical test split, and narrowband and wideband PESQ must not be compared to each other.

---

## Distribution

**How will the dataset be distributed?**
Via a public research archive (for example Zenodo or Hugging Face) with a DOI.

**Compiled dataset licence: CC BY-NC 4.0.**
Four of the five noise recordings are distributed by their holders under Creative Commons
noncommercial licences, so the compiled dataset is released for noncommercial use with
attribution. Commercial use of the audio is not permitted under this release.

**Clean speech (LibriSpeech).**
LibriSpeech is distributed under CC BY 4.0, so the clean targets may be redistributed with
attribution to Panayotov et al., 2015. CC BY content may be included inside a noncommercial
compilation.

**Noise recordings, per source.**
Each recording is credited to its holder, and these credits must be retained in any use.

| Noise | Recording holder | Basis for inclusion |
|---|---|---|
| rainfall | Sonatech, Inc. | CC BY-NC |
| lightning | H. Bass, R. Arnold, and A. Atchley, National Center for Physical Acoustics | CC BY-NC |
| hurricane | David Mann, Loggerhead Instruments | Written permission of the recordist to redistribute, with credit |
| bubble_curtain | JASCO Applied Sciences | CC BY-NC |
| large_commercial_ship | Thomas R. Kieckhefer | CC BY-NC |

The recordings were located through the Discovery of Sound in the Sea (DOSITS) archive, University
of Rhode Island Graduate School of Oceanography, which is acknowledged as the archive of origin.

**Are there other restrictions?**
LibriSpeech asks users not to attempt to identify individual speakers. The dataset licence is
noncommercial, as above.

---

## Maintenance

**Who maintains the dataset, and how can they be contacted?**
The maintainers (name and contact on release).

**Will it be updated?**
Planned future variants: a channel realistic version (absorption and multipath impulse
responses), a multichannel version enabling beamforming, and optional recording chain modelling.
Additional noise conditions may be added if further recordings are cleared for redistribution.
Versions will be released as distinct DOIs with a changelog.

**How are versions handled?**
Each release is versioned (v1.0 is the initial 100 speaker, 8 SNR, 5 noise build) with frozen
splits and seeds so that reported numbers remain comparable.

**How can others contribute or report issues?**
Via the archive's issue and contact channel listed on the record (to be filled in on release).
