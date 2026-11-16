# SEISGR4N — Technical Manual

**SEISGR4N** is a browser based instrument that transposes real-time seismic waveform MiniSEED data into continuous musification. 

By **SHOOK** · [@ALLSH00K](https://x.com/ALLSH00K) · [sos.allshookup.org](https://sos.allshookup.org)

> SEIS.MONSTER — Sweet Love For Planet Earth

---

## Contents

1. [Quick start](#1-quick-start)
2. [Design constraints](#2-design-constraints)
3. [Architecture](#3-architecture)
4. [Data pipeline](#4-data-pipeline)
5. [The strike: audification](#5-the-strike-audification)
6. [The SEISGR4N engine](#6-the-seisgr4n-engine)
7. [SEISSHIFT: transitions](#7-seisshift-transitions)
8. [Sections and the long-form arc](#8-sections-and-the-long-form-arc)
9. [Catalogue playback](#9-catalogue-playback)
10. [Instruments](#10-instruments)
11. [Audio graph](#11-audio-graph)
12. [The globe](#12-the-globe)
13. [Interface](#13-interface)
14. [URL state and snapshots](#14-url-state-and-snapshots)
15. [Browser support and fallbacks](#15-browser-support-and-fallbacks)
16. [Development and testing](#16-development-and-testing)
17. [Version history](#17-version-history)
18. [Credits](#18-credits)

---

## 1. Quick start

SEISGR4N is a single HTML file with no dependencies and no build step.

```bash
# any static server works; opening the file directly also works in most browsers
python3 -m http.server 8000
# then open http://localhost:8000/SEISGR4N.html
```

- **Audio starts on load** when the browser allows autoplay. Otherwise the play button pulses until the first click or keypress.
- **Master level starts at 0.8.**
- **The catalogue is on by default.** It shuffles through the M6.0+ catalogue, plays each earthquake for about three minutes, and doesn't repeat one until all have played.
- **The rail on the left** opens the pop-outs (Transport, Instruments, Knobs, Earthquake, Seismogram, Filters, View, Save, Info). There is no text on screen at rest; every pop-out can be closed.

> The current build file is `SE1STR0N1CA-V62.html`. Rename it to `SEISGR4N.html` (or `index.html`) for the repository.

---

## 2. Design constraints

These rules hold across the whole codebase and apply to any contribution.

| Constraint | Rule |
|---|---|
| Delivery | One self-contained HTML file. No external scripts, fonts or libraries. |
| JavaScript | ES5 on the main thread. ES6 only inside AudioWorklet source strings. |
| Graphics | WebGL1 / GLSL ES 1.00. A 2D-canvas fallback when WebGL is unavailable. |
| MiniSEED spine | `decodeSteimFrame`, `parseMiniSEED` and `fetchMiniSEED` are frozen. Do not modify them. |
| Visual canon | An orange UI palette; no text at rest; every pop-out can be closed. |
| Physics | All seismic wave propagation is drawn on or inside the globe, never outside it. |

---

## 3. Architecture

The file has five main parts:

```
SEISGR4N.html
├── <style>                  orange UI, pop-outs, rail, mobile layout
├── GLSL shaders             globe, field, plumes, monster, ring, points
├── <body>                   rail, pop-outs (markup only)
└── <script>  (ES5)
    ├── MiniSEED spine       decodeSteimFrame · parseMiniSEED · fetchMiniSEED   (frozen)
    ├── signature            magnitude / depth / entropy / boundary → S.*
    ├── audio core           AC, COMP, MASTER, LIMITER, ANA, REV
    ├── AUD                  the strike (audification)
    ├── PLUMES               mantle-plume swells
    ├── LEVEL · ARC · SHIFT  level matching, long-form arc, transitions
    ├── RINGEAR              ring chroma listener for non-GR4N instruments
    ├── GR4N · EngineGR4N    the granular arpeggiator
    ├── CATPLAY              catalogue sections
    ├── ENGINES[]            13 instruments
    ├── SE1SFEM              in-browser wavefield (spectral-element-inspired)
    ├── Globe                camera, LOD, layers, monster, ring
    ├── UI                   pop-outs, knobs, filters, persistence
    └── main loop            requestAnimationFrame → engines · CATPLAY · SHIFT · Globe
```

### Seeded randomness

All musical decisions go through seeded generators (`makeRng(key)`).

- **Per-event decisions** are keyed on the event ID.
- **Take-level decisions** add a per-session nonce (`GR4N.session`) and a take counter (`GR4N.count`).

As a result, every listen is different, but a single take is reproducible inside a session.

---

## 4. Data pipeline

| Source | Endpoint | Use |
|---|---|---|
| USGS FDSN event | `earthquake.usgs.gov/fdsnws/event/1/query` | The M6.0+ catalogue (CSV) and single-event detail (GeoJSON) |
| USGS summary feed | `earthquake.usgs.gov/earthquakes/feed/v1.0/summary/2.5_month.geojson` | Recent context seismicity |
| EarthScope FDSN dataselect | `service.earthscope.org/fdsnws/dataselect/1/query` | The waveform, as MiniSEED |

**Flow:**

1. Choose a station from a built-in list of global broadband stations, at a suitable distance (BHZ channel).
2. Request a window starting at the origin time. The default length is 630 s.
3. Decode the Steim-compressed MiniSEED into `S.data` (counts), with `S.trueRate` (usually 40 sps).
4. Build an 8 kHz playback buffer, `S.audioBuf`, used for the grains.
5. Compute the signature:
   - **Magnitude** maps to `S.mag` (0–1, from M6.0 to M9.5).
   - **Depth** maps to `S.depth` (piecewise: 0–35 km, 35–300 km, 300–700 km).
   - **Waveform entropy** maps to `S.entropy`.
   - **Plate-boundary class** (divergent, convergent, transform or intraplate) maps to `S.boundary`.

Live data is required. There is no bundled offline catalogue.

---

## 5. The strike: audification

Each earthquake opens with its own seismogram, sped up into hearing range. This follows the sos.allshookup.org auditory-display method.

| Layer | Speed-up | Filtering | Role |
|---|---|---|---|
| Bass | 0.4–0.9 × the BASS setting (default 100×, so 40–90×), drawn fresh each take | High-pass 16–20 Hz, low-pass 90–170 Hz (4th order) | The deep body of the rupture |
| Body | 276× | High-pass 20 Hz, plus a 0.5 Hz ground-domain high-pass to remove ocean microseisms | Detail and texture |
| Sub | A sine in A (27.5, 30.9, 34.6, 41.2 or 46.3 Hz) | Gliding down 3–8 semitones, lightly driven, low-pass 220 Hz | Felt more than heard |

**How the layers fit together:**

- Both recorded layers are resampled to the context rate with a Lanczos kernel, and their P-wave onsets are aligned.
- The sub follows a 160-point envelope taken from the bass layer.
- The strike bypasses the bus compressor and goes straight to `MASTER`, so the limiter and the recorder still see it.
- While the strike sounds, `ENGGATE` holds the instruments. They enter as it decays (`AUD.release`), and the seismogram sweep starts at that moment.
- The gold SEISMONSTER ruptures on the P onset of the bass layer.

**Every strike differs.** The bass speed-up, filters, length (3–5 s), sub pitch, glide, drive and level are drawn from `makeRng(session#strike#n#event)`. The Transport pop-out shows the current take, for example:

```
STRIKE · bass 88× + sub 46.3 Hz ↘ 3.2 st · 4.7 s · body 276× (microseism filtered) · 40 sps · P at 0.34 s
```

---

## 6. The SEISGR4N engine

SEISGR4N pairs two roles.

- **A GRAIN instrument** actually plays, as a sub-engine.
- **An ARP instrument** lends its voice to the arpeggio.

Each arpeggio note is also a cloud of grains cut from the real seismogram and pitched to the note.

### 6.1 Roles

| Role | Instruments |
|---|---|
| GRAIN | SEISROADS, SEIS0PN, SEISS0PHIE, SEISAUT3 |
| ARP | SEISB4RB, SEIST3RRY, SEISREICH, SEISKR4FT, SEISAPH3X, SEISB0C |

### 6.2 Choosing a pair

Each candidate pair is scored on five factors:

1. **Ideals:** distance from each instrument's ideal point in (magnitude, depth, cluster) space.
2. **Boundary prior:** ×2.5 for instruments suited to the plate-boundary class.
3. **Trace character:** impulsiveness, sustain and noise. For example, SOPHIE gets ×1.5 on impulsive traces.
4. **Pair table:** favoured pairs (ROADS + B4RB 1.3, 0PN + B0C 1.25, AUT3 + KR4FT 1.2, and so on) and penalised clashes.
5. **History:** the previous pair is excluded, and the last six are penalised.

### 6.3 Harmony and rhythm

| Parameter | Source |
|---|---|
| Chord | Plate boundary (convergent, divergent, transform or intraplate), snapped to a latitude-derived scale |
| Register (root, 82–330 Hz) | Depth: deeper quakes are lower |
| Range (1–3 octaves) | Magnitude |
| Subdivision (2, 3, 4 or 8 per beat) | Magnitude. The great quakes get 32nd-note bursts and rests. |
| Pattern | The boundary's pattern (up, down, up-down or Alberti) often, otherwise any |
| Chord changes | At the S-wave and surface-wave arrivals (a per-boundary progression) |
| Coda | Grains freeze on the arrival and the pattern follows the trace's contour |
| Grain pitch | `rate = note frequency / refHz`, where refHz is the trace's dominant frequency |

### 6.4 Variation ("never the same")

Each take varies the pattern, inversion, direction, tempo (×0.9–1.1), subdivision, octave range and register. If the resulting shape matches the previous take exactly, the pattern and inversion are moved on.

### 6.5 Knobs

| Knob | Range | Default |
|---|---|---|
| Arp rate × | 0.5–2 | 1 |
| Grain % | 0–100 | 55 |
| Cycle s (used only when the catalogue is off) | 0–120 | 31 |
| Spray % | 0–100 | 35 |

---

## 7. SEISSHIFT: transitions

SEISSHIFT shapes every change: a new movement, a new earthquake, or an instrument picked by hand. Each transition draws two to four of nine moves, fresh every time.

| # | Move | What it does |
|---|---|---|
| 1 | On the beat | The change lands on the bar after next. The bar before it is a fill: doubled speed, or two beats of rest. |
| 2 | Common-tone pivot | The new chord is transposed (−5…+6 semitones) to keep the most shared pitch classes, followed by one voice-led bar. |
| 3 | Tempo | Either a glide over about 6 beats, or a metric pivot that keeps the note length steady. |
| 4 | Filter handoff | The outgoing instrument's low-pass closes; the incoming one opens from the top. |
| 5 | Granular dissolve | The outgoing music freezes at its point in the trace and smears into a grain cloud. |
| 6 | Reversed preview | The next earthquake's bass layer plays backwards for 1–3 s, swelling into its own strike. |
| 7 | The quakes decide | Distance sets the length (a quake more than 90° away also gets 1–2 s of silence), a magnitude step lengthens the swell, and years apart lengthen the reverb tail. |
| 8 | Instrument entrances | See below. |
| 9 | Level matching | Each source has an analyser and a slow make-up gain toward a common loudness, limited to ±9 dB. |

**Entrances (move 8):**
- REICH: a second voice phases in and locks.
- B4RB: the sequence builds one note at a time.
- B0C: starts detuned and wobbly.
- KR4FT: downbeats first.
- T3RRY: a pulse before the pattern.
- APH3X: the filter rises.
- ROADS: thin to full.
- 0PN: an underwater swell.
- AUT3: a stuttered entry.
- S0PHIE: a hard cut on the bar.

The Earthquake pop-out's **SHIFT** line names the moves the last transition used.

---

## 8. Sections and the long-form arc

### 8.1 Sections (about three minutes per earthquake)

- **Length:** 165–195 s per section, drawn per section.
- **Sweep:** the seismogram is read once across the section, from the P wave to the coda (`CONFIG.playbackSpeed = 1 / (60 × SEC)`). The playhead holds on the coda instead of wrapping.
- **Movements:** four to six of uneven length. Each one is a new SEISGR4N take that enters through SEISSHIFT.
- **Envelope:**
  1. a build over the first 12%;
  2. a peak, placed between 50% and 70%;
  3. a release that thins out over the final 12%.

### 8.2 The long-form arc

A slow sinusoidal arc lasting 8–12 minutes, with its period and phase drawn per session, blends with the section envelope (45% arc, 55% section). It drives:

- grain-cloud density;
- arpeggio sparseness (dropped notes in the valleys);
- the arpeggio's low-pass, from about 500 Hz to 9 kHz;
- the balance between grain and arpeggio.

The Earthquake pop-out's **ARC** line shows the position, for example:

```
section 1:24 / 2:51 · movement 3/5 · dense 75%
```

---

## 9. Catalogue playback

| Setting | Behaviour |
|---|---|
| CATALOGUE | On by default. Moves to a new earthquake at the end of each section. |
| ORDER: SHUFFLE | Seeded per session. No earthquake repeats until every earthquake in the filtered catalogue has played, then the pass counter advances. |
| ORDER: DATE | In date order from 2000. |
| FILTERS | Year range (FROM/TO), minimum magnitude, depth and plate boundary (CONV/DIV/TRANS/INTRA) limit the pool. |

The section timer starts when the strike releases, not when the earthquake is selected. A slow waveform download delays the section, but doesn't shorten it.

---

## 10. Instruments

SEISGR4N ships 13 instruments. SEISFU©K and SEISX3N were retired in V59; old links that name them open on SEISGR4N.

| ID | Instrument | After | Character |
|---|---|---|---|
| `roads` | SEISROADS | Curtis Roads | Microsound granular clouds |
| `reich` | SEISREICH | Steve Reich | Phasing bell patterns |
| `one` | SEIS0NE | Brian Eno | Ambient tape-loop drift |
| `terry` | SEIST3RRY | Terry Riley | Pulse and modal loops |
| `kraft` | SEISKR4FT | Kraftwerk | Sequenced square-wave synth |
| `barb` | SEISB4RB | Caterina Barbieri | Resonant evolving sequences |
| `boc` | SEISB0C | Boards of Canada | Detuned, wobbly analogue |
| `aphex` | SEISAPH3X | Aphex Twin | Microtonal acid |
| `opn` | SEIS0PN | Oneohtrix Point Never | Hazy synth swells |
| `sophie` | SEISS0PHIE | SOPHIE | Hard, plastic transients |
| `autechre` | SEISAUT3 | Autechre | Generative, stuttered |
| `meta` | SEISMETA | — | Draws a different instrument character each take |
| `gr4n` | SEISGR4N | — | The granular arpeggiator (default) |

**Layering:** two instruments can play at once in slots A and B, with an equal-power crossfade. Per-instrument trims (`TRIM`) are applied before level matching.

---

## 11. Audio graph

```
instrument slot ─► slot LP/HP (handoff) ─► LEVEL ─► ENGGATE ─► ENGLP ─► COMP ─► MASTER ─► LIMITER ─► out
                                                                          ▲            ▲        └─► ANA (spectrum)
strike: bass + body + sub ────────────────────────────────────────────────┘ (to MASTER)
reversed preview · grain dissolve (SHIFT bus) ──────────────────────────────► MASTER
plume swells (PLGATE) ──────────────────────────────────────────────────────► MASTER
REV (3.2 s convolution) ─► REVGAIN ─► COMP
```

| Node | Role |
|---|---|
| `ENGGATE` | Holds the instruments while the strike sounds |
| `ENGLP` | Bus low-pass for the filter handoff on earthquake changes |
| `REVGAIN` | Raised temporarily for long time gaps between earthquakes |
| `ANA` | Feeds the spectrogram and RINGEAR |

---

## 12. The globe

| Feature | Detail |
|---|---|
| Camera | A quaternion arcball, flights between epicentres, pinch and wheel zoom |
| Earth spin | West to east, 0.5°/s (one turn in 12 minutes). Pauses while you drag and eases back in. Toggle: **SPIN**. |
| LOD | A surface patch sized to the view footprint for deep zoom |
| Epicentre pins | Point inward to the hypocentre depth |
| SE1SFEM | An in-browser wavefield (inspired by SPECFEM3D_GLOBE), sampled in the globe's vertex shader (texture unit 5). Waves travel only on and inside the globe. |
| P ray | Traced through an IASP91 P-velocity profile |
| Mantle plumes | 15 plumes and the two LLSVP domes. Each swells audibly as the surface wave reaches it; **DRONE BED** adds a constant drone. |
| SEISMONSTER | A single gold creature. It forms at each epicentre over about 1.2 s, ruptures on the strike's P onset over about 2.6 s, then disappears. |
| SEISGR4N ring | Always on. A pitch dial around the epicentre: θ is pitch class (clockwise from north), the radius is the octave. It shows chord points, note flashes and grain sparks, drawn as points only and kept at the same screen size when zooming. Under other instruments, **RINGEAR** derives the notes from a 12-bin chroma of the output and flashes on spectral-flux onsets. |

### View toggles

COAST, PLATES, SPIKES, HEAT, STATIONS, PATH, MONSTERS, LABELS, TRAILS, BLOOM, AUTO-ROTATE, SE1SFEM, SHAKEMAP, OUTWARD, RELIEF, HALO, PLUMES, X-RAY, SPIN, DRONE BED.

Defaults since V58.2: no lines drawn (COAST, PLATES and PATH are off). The family-A rays and the ring polygon were removed.

---

## 13. Interface

| Pop-out | Contents |
|---|---|
| Transport | Play/stop, master level, strike controls (STRIKE, BASS, BODY, microseism filter), CATALOGUE and ORDER |
| Instruments | The 13-instrument grid and the A/B layer |
| Knobs | The selected instrument's parameters |
| Earthquake | Event details, plus the GR4N pair, SHIFT moves and ARC position |
| Seismogram | A compact trace (300 px, bottom right) with drag-to-scratch |
| Filters | Year range, minimum magnitude, depth and plate boundary |
| View | Globe layer toggles and follow mode |
| Save | Snapshot export and recording |
| Info | About, credits and the link to sos.allshookup.org |

**Persistence:** pop-out positions, view toggles, filters, signature mode, strike settings and catalogue settings are stored in `localStorage` under a versioned key (currently `sos-v62`). Each version bump resets saved settings once.

**Mobile:** pop-outs dock to the bottom, and only one is open at a time.

**Reduced motion:** with `prefers-reduced-motion`, camera flights, shake and monster animation become near-instant.

---

## 14. URL state and snapshots

State lives in the URL hash:

```
#<instrument>?sig=<mode>&ev=<eventId>&b=<instrumentB>&xf=<0–100>&<knob>=<value>&b.<knob>=<value>
```

| Example | Meaning |
|---|---|
| `#gr4n?sig=classic&ev=usp000e8qf` | The default boot |
| `#roads?b=terry&xf=60&sig=classic&ev=us7000test` | ROADS layered with T3RRY at a 60% crossfade |
| `#compose=reich` | An alias for `#reich` |

**SAVE** writes a standalone HTML snapshot that reopens on the same earthquake, instruments and settings. Takes are seeded per session, so a snapshot reopens the setup rather than the exact music.

---

## 15. Browser support and fallbacks

| Situation | Behaviour |
|---|---|
| Autoplay blocked | The play button pulses until the first gesture |
| No WebGL | A 2D fallback: the seismogram becomes the main view (wide, with a spectrogram) and audio runs normally |
| No `StereoPannerNode` | Grains play centred |
| Waveform fetch fails | Status message. The catalogue moves on at the next section. |

Tested in headless Chromium (SwiftShader WebGL) on desktop and mobile viewports. Sub content around 28–46 Hz needs full-range speakers or headphones.

---

## 16. Development and testing

The release file is assembled by a Python build script. It takes the V55 source and applies anchored replacements plus module inserts (`audify`, `shift`, `gr4n`, `sfem`, `globe`, `ui`, `catalog`, `load`, plus the shaders, body and style).

**Rules for edits:**
- Keep the main thread ES5. The build runs an ES5 parse check on every script block.
- Never touch the MiniSEED spine.
- Keep every visual on or inside the globe.

**Regression suite** (Playwright with mocked USGS/EarthScope responses):

| Check | Asserts |
|---|---|
| Desktop and mobile boot | No visible text at rest, no page errors, deep zoom works |
| Strike flow | Hold → release → run; SAVE snapshot parses as ES5 and reopens |
| Snapshot boot and no-GL | Fallback plays audio with no errors |
| Sections | Movements fire at the planned boundaries; the turnover moves to a new event |
| SEISSHIFT | Moves are logged per transition; retired instrument hashes redirect |
| Ring and spin | Ring point counts under several instruments; the centre longitude drifts west |

Console logs are prefixed `[SOS]` (strike takes, SEISGR4N takes, SEISSHIFT moves).

---

## 17. Version history

| Version | Changes |
|---|---|
| V55 | SE1STR0N1CA base: 13 artist instruments, MiniSEED pipeline |
| V57 | Full globe; SEISMETA as the 14th instrument |
| V57.1 | SE1SFEM phase 1: the wavefield is confined to the globe |
| V57.2 | Gold SEISMONSTER only; audification strike; 15 mantle plumes |
| V57.3 | Hum fixes (plume drones, microseism filter, gating) |
| V58 | SEISGR4N granular arpeggiator as the 15th instrument; starts on load |
| V58.1 | Rays removed; catalogue shuffle; never-the-same takes; master at 0.8; arpeggio ring |
| V58.2 | No lines; the monster forms and ruptures; a deep, varied strike sub |
| V59 | SEISFU©K and SEISX3N retired; SEISSHIFT (nine transition moves); long-form arc |
| V60 | Three-minute sections with movements and a build/peak/release envelope |
| V61 | Earth spin; the SEISGR4N ring always on, with RINGEAR for other instruments |
| V62 | Compact bottom-right seismogram |

---

## 18. Credits

- **Concept, direction and seismic sonification practice:** SHOOK (D.V. Rogers).
- **Seismic data:** USGS (event catalogue) and EarthScope (formerly IRIS) FDSN web services and the global broadband station networks it serves.
- **Wavefield method:** inspired by SPECFEM3D_GLOBE (Komatitsch, Tromp et al.). SE1SFEM is an independent, simplified browser implementation, not a port.
- **Velocity model:** IASP91 (Kennett & Engdahl, 1991).
- **Instruments:** named in tribute to the artists listed. No samples or recordings of their work are used; all sound is synthesised or made from seismic data.

**License:** MIT
