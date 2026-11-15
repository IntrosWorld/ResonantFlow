# ResonantFlow

Physical-model synthesizers (piano, guitars, drums) play the notes. A small learned stage, trained on real recordings,
gives them the tone of real instruments, amplifiers and pedals.

The learned stage is the **resonant flow** (`PhysicalToneLab/runs/rnf_resonant_amp_v6/models.py`, class `SpectralFlow`,
3,696 parameters): eight weight-tied steps on harmonic envelopes, where harmonics may only interact in
frequency-conserving ways (i ± j = k, i ± j ± l = k) and the interaction strengths come from one small network that
reads frequency. Amplifiers and pedals use the full flow. Piano, guitars and drums use a restricted form that only sets
gain and decay per note and cannot create new frequencies.

## 1. Folders

| Folder | Contents |
| --- | --- |
| `PhysicalToneLab/runs/rnf_resonant_amp_v6/` | The architecture, amplifier training, baselines |
| `PhysicalToneLab/runs/rnf_instruments_v7/` | Piano, steel guitar, clean electric guitar, bass, drums, crash; pedals and chorus in `tones/` |
| `PhysicalToneLab/runs/rnf_song_ascend/` | Full-song renderer, the example tab, example mixes and stems |
| `PhysicalToneLab/runs/rnf_port_study/` | Speed measurements |
| `grand_piano_improved/`, `acoustic_guitar_fea/`, `electric_guitar_lead/` | The physical-model engines |

Keep this layout: the scripts find each other by relative path.

## 2. Setup

- Python 3.10 or newer (developed on 3.13, Windows 11). CPU only; no GPU is needed.
- About 16 GB of RAM. Run one render at a time.

```bash
pip install torch numpy scipy soundfile numba
```

Commands are for a Bash shell (Git Bash on Windows, or Linux/macOS). In every new shell, first run:

```bash
export PYTHONDONTWRITEBYTECODE=1 PYTHONIOENCODING=utf-8; PY=python
```

If `python` is not the interpreter with the packages, set `PY` to its full path.

All trained checkpoints are included, so nothing has to be trained to render audio.

## 3. Render a MIDI file (piano)

Works straight after cloning.

1. Copy your file into `grand_piano_improved/midi/` (for example `mysong.mid`). Demo files are already there.
2. Run, giving the file name, the number of seconds to render, and a label for the output:

```bash
cd PhysicalToneLab/runs/rnf_instruments_v7
```
```bash
PIANO_LOW_CKPT=piano_low_attack PIANO_AIR_DB=12 PIANO_AIR_HZ=2000 PIANO_BASS_DB=15 PIANO_BASS_HZ=300 $PY perform_midi2.py mysong.mid 60 H
```

3. Output: `audition/mysong_60s_H.wav` (stereo, 44.1 kHz, 24-bit).

- Every note in the file is played as piano.
- Give the file **name**, not a path.
- Speed: the first 15 s of `demo10.mid` (127 notes, 20 different ones) took 21 s on a laptop CPU. Time grows with the
  number of different notes, about one second each.

## 4. Render a Guitar Pro file (full band)

The renderer is `PhysicalToneLab/runs/rnf_song_ascend/render_song.py`. It reads a Guitar Pro 7/8 file (`.gp`), renders
one stem per instrument, and mixes them.

### 4.1 One-time preparation

Piano and drums need nothing. The steel-guitar and electric-guitar stems need two data files that are too large for the
repository; these commands download public recordings and rebuild them:

```bash
cd PhysicalToneLab/runs/rnf_instruments_v7
```
```bash
$PY fetch_guitarset.py
```
```bash
$PY guitar_flow.py prep
```
```bash
$PY fetch_egfx.py
```
```bash
$PY electric_clean_flow.py prep
```

### 4.2 Render the included tab

`tab/Ascend.gp`, first 120 seconds. Run the commands one after another, not in parallel:

```bash
cd PhysicalToneLab/runs/rnf_song_ascend
```
```bash
$PY render_song.py piano
```
```bash
$PY render_song.py steel
```
```bash
$PY render_song.py guitars
```
```bash
$PY render_song.py drums
```
```bash
$PY render_song.py mix
```

Output: `ascend_first_120s_full_mix.flac` (stereo, 48 kHz) and one file per instrument in `stems/`.

**The `guitars` stage does not run from this repository alone.** It reads the amplifier's training recordings from a
folder `NeuralInstrumentStudio_Experiments/energy_v3/data/captured_amp` next to `PhysicalToneLab/`, which is not
included. Without it, skip `guitars` and remove `guitar_lead` and `guitar_rhythm` from the `plan` dictionary in `mix()`.
The guitar stems of the included tab are provided as audio in `stems/`.

### 4.3 Use your own Guitar Pro file

Edit three things in `render_song.py`:

1. **File:** replace `tab/Ascend.gp` with your file under the same name, or change `GP = HERE / 'tab/Ascend.gp'`.
   Only `.gp` (Guitar Pro 7 and 8) is read; export older formats to `.gp` first.
2. **Length:** `SECONDS = 120.`
3. **Track numbers** (they start at 0, in Guitar Pro's order):

| Stage | Where it is set | Included tab |
| --- | --- | --- |
| `piano()` | `for t in (1, 2)` and `notes_of(2)` (left hand) | tracks 1, 2 |
| `steel()` | `notes_of(4)` | track 4 |
| `guitars()` | `(('guitar_lead', 0), ('guitar_rhythm', 3))` | tracks 0, 3 |
| `drums()` | `parse_drum_track(GP, 5)` | track 5 |

If your file has no track for a stage, skip that stage and remove its entry from `plan` in `mix()`.

Options: `PIANO_PEDAL=0` before the piano command turns the sustain pedal off. Mix levels and panning are the `plan`
dictionary in `mix()`.

## 5. Datasets

The real recordings used for training are public but are not stored in this repository. Each one is downloaded by its
own script into `PhysicalToneLab/runs/rnf_instruments_v7/data/`. Run the scripts from `PhysicalToneLab/runs/rnf_instruments_v7/`.

| Dataset | Used for | Where it comes from | Licence | Download command |
| --- | --- | --- | --- | --- |
| University of Iowa Musical Instrument Samples (MIS) | Piano, crash cymbal | https://theremin.music.uiowa.edu/MIS.html | Free to use, no restrictions | `bash get_piano.sh` and `$PY fetch_crash.py` |
| GuitarSet | Steel-string guitar | https://zenodo.org/records/3371780 | CC BY 4.0 | `$PY fetch_guitarset.py` |
| EGFxSet | Clean electric guitar, distortion pedals, chorus | https://zenodo.org/records/7044411 | CC BY 4.0 | `$PY fetch_egfx.py` and `$PY tones/fetch_chorus.py` |
| IDMT-SMT-Bass | Electric bass | https://zenodo.org/records/7188892 | CC BY-NC-ND 4.0 (research use, no redistribution) | `$PY fetch_bass.py` |
| IDMT-SMT-Drums V2 | Kick, snare, hi-hat | https://zenodo.org/records/7544164 | CC BY-NC-ND 4.0 (research use, no redistribution) | `$PY drum_flow.py fetch` |

- The scripts download only the files the project uses (for example 48 of GuitarSet's 180 solo takes, and the
  fingerstyle single notes of the bass set), not the whole archives.
- The amplifier's training data is not public: it is a synthesised guitar signal passed through a neural capture of a
  real amplifier, made for this project. The harmonic envelopes extracted from it are included
  (`rnf_resonant_amp_v6/envelopes.npz`), which is enough to retrain and test every amplifier model.
- Cite the datasets if you publish results, and do not redistribute the two IDMT sets or files derived from them.

**To train an instrument from scratch**, download its dataset, then run `prep`, `fit` and `check`. For example, the
steel guitar:

```bash
$PY fetch_guitarset.py
```
```bash
$PY guitar_flow.py prep
```
```bash
$PY guitar_flow.py fit
```
```bash
$PY guitar_flow.py check
```

## 6. Training and evaluation

Run inside the folder in each heading. The usual order is `prep` (build pairs) → `fit` (train) → `check` (held-out
numbers) → `render` (audio). `check` and `render` work with the included checkpoints; `prep` needs the raw recordings,
which the fetch scripts download.

### Amplifier (`PhysicalToneLab/runs/rnf_resonant_amp_v6/`)

```bash
$PY train.py test
```
```bash
$PY report.py
```

`train.py test` prints the test scores of every model; `report.py` draws the chart. To retrain everything:
`bash run_all.sh`. One model: `$PY train.py fit <arm> <seed> [steps] [lr]` with arm `resonant`, `shuffled`,
`wiener_hammerstein`, `blackbox_small`, `blackbox_large` or `blackbox_canonical`.
`extract.py` and `render_audio.py` need the amplifier recordings folder named in section 4.2 (not public, see section 5).

### Pedals and chorus (`PhysicalToneLab/runs/rnf_instruments_v7/tones/`)

```bash
$PY tones.py test
```
```bash
$PY cond.py test
```

Retrain: `$PY ../fetch_egfx.py`, then `$PY tones.py extract <tone>` and `$PY tones.py fit <tone> resonant <seed>`
(tones: `BluesDriver`, `TubeScreamer`, `RAT`), or `bash run_all.sh` for everything. Chorus: `$PY fetch_chorus.py`, then
`$PY chorus.py fit`, `test`, `render`.

### Instruments (`PhysicalToneLab/runs/rnf_instruments_v7/`)

| Instrument | Script | Stages | Data first |
| --- | --- | --- | --- |
| Piano | `piano_flow.py`, `piano_decay.py`, `piano_low.py` | `prep`, `fit`, `check` | `bash get_piano.sh` |
| Steel guitar | `guitar_flow.py` | `prep`, `fit`, `check`, `render` | `$PY fetch_guitarset.py` |
| Clean electric | `electric_clean_flow.py` | `prep`, `fit`, `check`, `render` | `$PY fetch_egfx.py` |
| Bass | `bass_flow.py` | `prep`, `fit`, `check`, `render` | `$PY fetch_bass.py` |
| Kick, snare, hi-hat | `drum_flow.py` | `fetch`, `prep`, `fit`, `check`, `render` | built in (`fetch`) |
| Snare (retrained) | `drum_snare_v2.py` | `prep`, `fit`, `check` | uses the drum data |
| Crash | `crash_flow.py`, then `crash_ring.py` | `prep`, `fit`, `check`, `sample` | `$PY fetch_crash.py` |

Example:

```bash
$PY drum_flow.py check
```

## 7. Results (held-out data)

| What | Baseline | With the learned stage |
| --- | ---: | ---: |
| Amplifier, harmonic error (plain network 0.393, same network with the rule broken 0.339) | 0.810 | 0.090 |
| Blues Driver pedal | 0.638 | 0.030 |
| RAT pedal | 0.620 | 0.052–0.059 |
| Tube Screamer pedal (ties the baselines) | 0.380 | 0.052–0.061 |
| Steel guitar, attack error | 7.0 dB | 5.3 dB |
| Clean electric, error at 1 s | 8.9 dB | 6.6 dB |
| Bass, error at 1 s | 19.5 dB | 6.4 dB |
| Snare, attack error | 15.4 dB | 6.1 dB |
| Crash, band error | 8.5 dB | 2.9 dB |

Full logs, including failed attempts, are in the README of each run folder.

## 8. Where the audio is

| What | Path (under `PhysicalToneLab/runs/`) |
| --- | --- |
| Full-song mixes | `rnf_song_ascend/ascend_first_120s_full_mix_v*.flac` |
| Stems and single drum pieces | `rnf_song_ascend/stems/`, `rnf_song_ascend/drum_samples/` |
| Instruments, physical model alone vs with the learned stage | `rnf_instruments_v7/audition/` (`*_1_engine_only`, `*_2_flow`) |
| Pedals: clean, real pedal, our model | `rnf_instruments_v7/tones/audition/` |
| Amplifier | `rnf_resonant_amp_v6/audition/` |

## 9. Limits

- A MIDI file is played by the piano only. A Guitar Pro file needs its track numbers set by hand.
- The song renderer has no bass stage; electric guitars use one trained high-gain amp.
- Rendering is offline, not real time.
- Mix levels, rooms, the piano's bass and air settings and the sustain-pedal rule are hand settings, not learned.
- The amplifier was trained on a neural capture of a real amp with a synthesised guitar input.
- Raw recordings are not included; some datasets are research-use only and must not be redistributed.
