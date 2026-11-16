# S3D — Audit and Redevelopment Plan

16 November 2026 · for SHOOK  · Stage 0 deliverable, no code

---

## 1. What this is

S3D works, but it is hard to change safely: years of patches have left leftover code, several parts fighting over the same settings, and four separate clocks. This plan records how S3D sounds today, rebuilds it tidily so it behaves the same, and only then improves the sound and the picture.

This document contains no code. It is the Stage 0 deliverable from the brief: an audit of `index.html` plus a plan to approve, change or reject.

**How the audit was done.** It is a structural scan of the 7,269-line file, not a line-by-line read. I mapped the section banners and searched for sweep-speed writes, loops and timers, engine definitions, transition code, randomness, and audio and graphics node creation, then read the main loop and the engine registry. Items marked *to confirm* need a closer read before Stage 1.

**In plain words.** Think of S3D as a LEGO castle built on for years. There are leftover bricks nobody uses, too many bosses giving orders, and one sneaky problem where a broken download plays a pretend earthquake. The plan is: photograph the castle, rebuild it neatly, then make it better.

---

## 2. Audit findings

The file is 624 KB. About 160 KB of that is four inline map-data arrays (lines 1618–1622). The banners show code from V3, V5, V6, V8, V27/28, V32, V39, V48, V57, V57.2, V58 and V59 all living together, which is more layers than the brief's seven.

### 2.1 Leftover and duplicated code

| Item | Where | Note |
|---|---|---|
| Retired engines `EngineFUCK` and `EngineX3N`, plus the GENDYN worklet | Lines 2179, 2543 | FU©K's banner calls it the "chassis standard", so other engines may borrow its helpers. To confirm before removal. |
| Two scratch engines | Lines 6835, 6854 | V27/28 and V32 versions |
| Two visualizers | Lines 3929, 4027 | One is probably the no-WebGL fallback. To confirm. |
| Three waveform fetchers | Lines 867, 913, 7100 | Only `fetchMiniSEED` (913) is frozen |
| Stale comments and logs | Line 5049 and elsewhere | The catalogue banner still says "one event every 31 s"; 42 console calls, some prefixed `[FuckButtons]` |

### 2.2 Too many bosses

- **Sweep speed has four writers.** The signature code sets `CONFIG.playbackSpeed` on lines 1739 and 1742. The catalogue sets it on 5092 and again on 5123 every tick, "whatever the signature mode set". The conflict is settled by overwriting.
- **Four clocks.** Three things drive the music's `advance()`: the animation loop (line 6989), the worklet clock, and a 40 ms background timer. The globe runs its own separate animation loop (line 6333), so picture and sound do not share a clock today.
- **Transitions have two owners.** The pivot and metric-pivot logic lives inside SEISGR4N (lines 4830–4876), while SEISSHIFT (line 4676) logs the same move names.
- **Load-order guards.** Checks such as `typeof SHIFT !== 'undefined'` show modules reaching into each other.
- **8 kHz is hard-coded twice**, in `buildAudioBuf` (line 2024) and in the grain pitch reference (line 4910). Stage 2 must change both together.

### 2.3 The pretend-earthquake problem

`parseMiniSEED` catches parse errors and returns a made-up 8,192-sample trace with random noise. A corrupt download therefore plays a fake earthquake, against hard constraint 8 (live data only). The function is frozen, so the fix has to sit outside it: detect that case, reject it, and let the catalogue move on. This needs approval.

### 2.4 What tires a phone (likely; not yet measured)

- **A new sound-maker per note:** 37 oscillator and 11 buffer-source creation sites, heaviest in dense arpeggio and grain passages.
- **SE1SFEM:** stepped on the CPU with a texture upload each frame.
- **Two animation loops**, plus bloom buffers and the deep-zoom patch.
- **Start-up:** parsing the 79 KB plate-class array.
- **`ScriptProcessor`** in two places, which is audio work on the main thread.

---

## 3. Proposed layout of the single file

One file, eleven modules in a fixed order, and each shared setting has exactly one owner. A module may read from modules above it but never write into them.

| # | Module | Job | Owns |
|---|---|---|---|
| 1 | Spine | MiniSEED decoding, copied byte for byte | Nothing changes here |
| 2 | Data | Fetching, station choice, validation, map data | The current trace |
| 3 | Clock | Audio time is the only clock; one tick | Time |
| 4 | Score | A seeded plan for each section | Sweep speed, movements, chord events, arc |
| 5 | Audio core | Mix graph, strike, plumes | The mix |
| 6 | Engines | 13 instruments behind one interface (`start`, `stop`, `tick`, `onData`, `movement`) | Their own voices |
| 7 | SHIFT | Every transition | Transition notes and moves |
| 8 | Catalogue | Which earthquake is next | The current event |
| 9 | Globe and SE1SFEM | Drawing; reads Clock and Score, never writes | The camera |
| 10 | UI | Rail, pop-outs, URL state, snapshots | Settings |
| 11 | Harness | Hidden render mode | Nothing at run time |

**The key idea is the Score.** Each section becomes one plain data object, built from the seed, that lists when movements, chord changes and wave arrivals happen. The music reads it now. The globe reads it in Stage 5, which is how picture and sound end up sharing one clock without extra work.

---

## 4. Staged plan

The plan follows the brief's stages with one change of order: the render harness is built first, on today's file, so there is a "before" recording to compare the rebuild against. Each stage ends with a working `index.html` and a gate.

| Stage | What happens | Gate |
|---|---|---|
| 0.5 Harness | Add the hidden render mode to the current file. Render three reference earthquakes of different boundary types and magnitudes. | Reference WAVs and loudness numbers filed |
| 1 Rebuild | New tidy file in the layout above. Same behaviour. Leftovers removed. | All 12 acceptance checks pass; output matches the references |
| 2 Sound | Better grains, better arpeggio voices, steady strike loudness, one mix chain | Harness targets met; SHOOK listens and approves |
| 3 Composition | Session key plan, returning motifs, more data mapped | SHOOK approves the musical changes |
| 4 Stations and space | Three to six stations per quake, placed by bearing | Falls back to one station; no delay before the first strike |
| 5 Image | Timed wave fronts, rupture, camera moves, phone quality steps | Steady frame rate on a mid-range phone |
| 6 Extras | Breaking-quake mode, video export | Each needs approval |

### 4.1 Stage 0.5 and 1: proving "same behaviour"

- **Two tests, not one.** Compare loudness numbers, and also compare the `[SOS]` decision logs. For a fixed seed, the old and new builds should log the same pair, chord, pattern, movement times and SHIFT moves.
- **Suggested tolerance:** ±0.5 LU integrated loudness, ±1 dB per octave band, identical decision log.
- **Injectable seed.** The session nonce must be settable from the URL, or the harness cannot repeat a take.
- **Audio time only.** Nothing musical may depend on animation frames or timers, or it cannot be rendered offline.

### 4.2 Stage 2: sound

- **Grains:** resample straight from the native 40 samples per second to the playback rate with the Lanczos kernel the strike already uses, skipping the 8 kHz buffer. Run the overlap-add in an AudioWorklet, which is lighter on phones. Keep today's path as the fallback.
- **Voices:** one synthesis model per ARP character, for example an FM bell for REICH, a resonant filter sequence for B4RB, slide and accent for APH3X, wow and flutter for B0C. These are characters only, never imitations (hard constraint 10).
- **Strike:** the layers are pre-rendered, so measure each take's loudness before it plays and scale it. Variation stays in character, not level.
- **Mix:** replace the live ±9 dB level matching with fixed calibration measured by the harness, plus one shared reverb.
- **A gap in the −16 LUFS target:** LUFS weighting nearly ignores 28–46 Hz, so the sub can use up the peak headroom while barely registering. Add a separate 20–60 Hz target.

### 4.3 Stage 3: composition

- **Key plan:** the catalogue is shuffled, so harmony cannot simply "travel". Give each region or boundary system a home key, a fixed tonal map of the Earth. Modulation distance then mirrors geographic distance, and the existing common-tone pivot does the moving.
- **Motifs:** store each section's opening contour by region. Bring it back inverted, stretched or re-registered when a later quake lands there.
- **Focal mechanism:** USGS event detail usually carries moment-tensor nodal planes, and rake gives thrust, normal or strike-slip. Fall back to boundary class when absent. Field availability is to confirm.
- **Aftershocks:** one extra count query per event, fetched in the background so it never delays the strike.

### 4.4 Stage 4: stations and space

- Fetch the primary station first and the others during the strike and first movement. They arrive later in real time anyway.
- Pan each station by its bearing from the epicentre. Binaural mode can use the browser's built-in HRTF panner, which needs no extra files.

### 4.5 Stage 5: image

- Drive the wave fronts from the same sweep playhead that reads the seismogram. P, S and surface fronts then land exactly when the chord changes do.
- Scale rupture length from magnitude with standard empirical relations, and take its orientation from the nodal planes.
- Phone quality steps, in order: pixel ratio, bloom, wavefield resolution, plume detail.

---

## 5. Decisions needed and risks

Nothing gets built until these seven are answered. The first four block Stage 0.5 and Stage 1.

- [ ] **Pretend earthquakes:** approve rejecting the synthetic fallback trace, outside the frozen spine?
- [ ] **Harness and ES5:** `OfflineAudioContext` is promise-based. Accept that on the main thread for the hidden render mode only?
- [ ] **Order:** approve harness first, rebuild second?
- [ ] **Closer read:** approve reading the two *to confirm* regions (the FU©K chassis and the visualizers, roughly 1,000 lines) before Stage 1?
- [ ] **Tolerance:** accept ±0.5 LU, ±1 dB per octave band and an identical decision log as "same behaviour"?
- [ ] **Sub target:** add a separate 20–60 Hz loudness target beside −16 LUFS?
- [ ] **Key plan:** tonal map of the Earth, or should each key follow the previous quake?

### Risks

| Risk | Why it matters | How the plan handles it |
|---|---|---|
| The rebuild changes the music without anyone noticing | The takes are seeded and never repeat, so listening alone cannot prove sameness | Fixed seeds, reference renders and decision-log comparison |
| Removing FU©K breaks other engines | It may be the shared chassis | Confirm by reading before deleting; move shared helpers out first |
| Offline rendering misses timer-driven behaviour | 22 timeouts and 4 intervals exist today | Stage 1 moves all musical timing onto audio time |
| The performance findings are guesses | Nothing has been measured on a phone yet | Measure before Stage 5 sets its frame budget |
| The audit missed something | It was a scan, not a full read | Each module gets a full read when its stage begins |
