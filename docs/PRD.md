# Product Requirements Document: DJ Rhythm Trainer

## 1. Summary

A mobile rhythm game that turns a DJ's own downloaded music library into practice material. Players import audio files already saved on their phone; the app analyzes each track (BPM, key, waveform, and frequency-band onsets for bass/drums/beats/vocals) and turns that analysis into a Guitar Hero–style note chart. By playing through their own tracks, DJs build a felt sense of a track's structure, timing, and energy changes — useful for planning mixes and transitions.

## 2. Goals

- Let a DJ import music already on their device (no streaming, no catalog, no uploads) and play a rhythm game built from that track's actual audio content.
- Surface useful track intel passively: BPM, key (if tagged), waveform, artwork, and metadata.
- Make the gameplay loop teach something real — timing accuracy against the track's actual bass/drum/vocal onsets, not a generic metronome.
- Ship as a fully offline, on-device experience. No account, no backend, no audio ever leaves the device.

## 3. Non-Goals (MVP)

- No streaming service integration (Spotify/Apple Music API playback).
- No multiplayer, leaderboards, or social sharing.
- No true ML source separation (stems). MVP uses frequency-band onset detection as a proxy for bass/drums/vocals.
- No DRM-protected file support (e.g., Apple Music DRM tracks) — only unprotected local files (MP3, M4A/AAC, FLAC, WAV, OGG).
- No cloud sync/backup of scores or libraries.

## 4. Target User

Working and hobbyist DJs who already keep a library of purchased/ripped tracks on their phone and want a faster, more game-like way to internalize a track's structure (where the drop is, where vocals come in, where the breakdown starts) than passive listening.

## 4a. Competitive Analysis

A market scan (iOS/Android stores + web) found **no existing app that combines a rhythm game with DJ-focused ear-training on the player's own local library.** The space splits into two adjacent categories that each cover only part of this concept, which is the core opportunity/differentiator.

### Category A — Rhythm games that use your own music (closest on *gameplay*)

| App | Platform | How it works | What's worth borrowing |
|---|---|---|---|
| **Cytoid** | iOS/Android | Community-authored charts; strong audio-sync tech | **Latency calibration & one-time device sync setup** — essential for any timing game |
| **Beatstar** | iOS/Android | Licensed songs only, hand-authored charts with tap/slide/hold notes | Polished note-streaming feel; **variety of note types** (tap/hold/slide), not just single taps |
| **Audiosurf / Beat Hazard / Symphony** | mostly PC/older | Auto-generate gameplay from any song the user loads | Proves **auto-chart-generation from arbitrary local audio is viable** — validates our core premise |

None of these target DJs, and the licensed-music ones (Beatstar) are catalog-locked because they had to license their tracks — a constraint we sidestep entirely by only ever using the user's own local files and never transmitting them.

### Category B — DJ analysis tools (closest on *features*, but no gameplay)

| App | What it does | What's worth borrowing |
|---|---|---|
| **Mixed In Key** | Industry-standard BPM + **key detection** (Camelot notation) | Confirms key is valuable to DJs; tags written by this tool let us read key for free |
| **BPM Analyzer** (iOS) | BPM detection + multiple **waveform styles** (bars, wave, circular, spectrum, oscilloscope) | Validates appetite for rich waveform visualization — our RGB waveform is a novel extension of this |
| **DJ.Studio** | AI beatmatching; visually breaks down transitions for *learners* | Closest in *intent* (teaching DJs) but it's a mixing tool, not a game |

### Conclusions folded into this PRD
1. **Latency calibration is now an MVP requirement** (Phase 3), not a nice-to-have — borrowed from Cytoid. A timing game is unplayable if input/audio/display latency isn't calibrated per device.
2. **Marker system is designed to support hold/sustain (and later slide) note types**, borrowed from Beatstar — full hold notes may land post-MVP, but the data model won't have to be reworked to add them.
3. Our **RGB frequency-band waveform** is already more novel than competitors' waveform views, confirming the visualization direction.

## 5. Platform & Tech Stack

**Platform:** Cross-platform via React Native (Expo bare/dev-client workflow — the managed workflow won't support the native modules this app needs).

**Why bare/dev-client, not managed Expo:** arbitrary folder access and on-device audio DSP both require native modules not available in Expo Go.

| Layer | Choice | Notes |
|---|---|---|
| App framework | React Native + TypeScript | Single codebase, iOS + Android |
| Navigation/UI | React Navigation, Reanimated + Skia (or React Native GL) | Reanimated/Skia needed for smooth 60fps falling-marker rendering and RGB waveform painting |
| Local storage | SQLite (via `expo-sqlite` or WatermelonDB) | Stores track library, cached analysis results, scores |
| File access (Android) | Storage Access Framework via a native/community module (e.g. `react-native-scoped-storage`) | User grants a folder; app gets persistable URI permission |
| File access (iOS) | `UIDocumentPickerViewController` (folder mode) + security-scoped bookmarks | iOS is more restrictive — see Risks |
| Metadata/artwork extraction | `music-metadata` (JS, reads ID3/MP4/FLAC/Vorbis tags from a buffer) | Reads artist, title, album, artwork, key (if tagged via `TKEY`/`initialkey`), BPM tag if present |
| Audio decode + DSP | Custom native module (Swift/Kotlin) wrapping platform decoders (AVAudioEngine / MediaCodec) feeding an FFT-based analysis pipeline | Needed for BPM detection, onset detection, and waveform peak extraction — not feasible in pure JS at acceptable speed |
| Playback | `react-native-track-player` or a thin native player | Needs frame-accurate position reporting to drive the note chart |

## 6. Core Features

### 6.1 Library Import
- User grants the app access to one or more folders on their device.
- App scans the folder(s) for supported audio files (MP3, M4A/AAC, FLAC, WAV, OGG).
- Each new file is queued for a one-time **analysis pass** (see 6.3). Results are cached locally keyed by file path + hash, so re-scans are instant unless a file changed.
- Library screen lists tracks with artwork thumbnail, title, artist, BPM, and key.

### 6.2 Track Metadata Display
- Album/track artwork (from embedded tags; fallback to a generic placeholder if absent).
- Artist name, track title, album.
- **BPM — kept for MVP.** Detected from audio via the analysis pipeline (mature, low-risk DSP), or read from the embedded tag when present and trusted. Common in DJ libraries as a tag written by Serato/Rekordbox/Mixed In Key.
- **Musical key — read-from-tag only for MVP.** If a key tag is present (`TKEY`/`initialkey`/Mixed In Key Camelot tags), display it as-is. If untagged, show **"Unknown"** — audio-based key computation (chroma analysis) is a separate, lower-accuracy DSP feature deferred to post-MVP (see Roadmap Phase 5). This gets real key values for most DJ tracks (which are typically tagged) without taking on the risky part.
- Duration, sample rate/bitrate (secondary info, e.g. an "info" expand panel).

### 6.3 Audio Analysis Pipeline (runs once per track, cached)
1. **Decode** the file to PCM via the native module.
2. **Waveform extraction:** downsample amplitude peaks per time window, split into low/mid/high frequency energy per window (via FFT) to drive the RGB waveform (see 6.4).
3. **BPM detection:** onset-strength/autocorrelation-based tempo estimation (standard beat-tracking DSP approach).
4. **Onset/band detection (marker generation):** per analysis window, run FFT and bucket energy into the 4 lane types (matching the color system in 6.4):
   - **Low (Bass)** — low-frequency band onsets (~20–250 Hz)
   - **Mid** — mid-frequency band onsets (~250 Hz–3.5 kHz)
   - **High** — high-frequency/percussive transient onsets (~3.5 kHz+, spectral flux spikes)
   - **Vocals** — mid-band onsets with vocal-formant/harmonic characteristics (detected separately from generic Mid onsets)
   Each detected onset above a confidence threshold becomes a **marker** with a timestamp, lane/type, intensity, and a **confidence score**. The full set of detected onsets is stored; **how many are surfaced as playable markers is decided at play-time by the selected difficulty (see 6.8)** — so one analysis pass serves all three difficulties.
5. Results (waveform data, BPM, key if tagged, marker timeline) are written to local storage as a compact JSON/binary blob per track.

This is a **precompute-once, play-from-cache** design — no real-time DSP during gameplay, which keeps the game loop simple and battery-friendly.

### 6.4 RGB Waveform Display & Unified Color System
- Full-track waveform rendered with color channels mapped to frequency bands per time slice, so a glance at the waveform shows where the track is heavy in bass vs. mids vs. highs.
- **Shared color language (key design principle):** the waveform band colors and the gameplay marker/lane colors are the *same palette*, so the player learns one color code and it reinforces across both surfaces. When they see "red is loud" in the waveform, red markers in the game mean the same thing.

| Band / Lane | Color | Waveform channel | Frequency proxy |
|---|---|---|---|
| **Low (Bass)** | Red | R | ~20–250 Hz |
| **Mid** | Green | G | ~250 Hz–3.5 kHz |
| **High** | Blue | B | ~3.5 kHz+ |
| **Vocals** | Accent (e.g. white/gold — TBD in design pass) | overlay highlight, not a raw RGB channel | mid-band *harmonic* content |

- **The 3-vs-4 resolution:** RGB is only 3 channels, but we have 4 lanes. Low/Mid/High map cleanly to R/G/B. **Vocals are special** — they're detected from *harmonic/formant* characteristics within the mid band, not a separate frequency channel, so a raw RGB waveform can't give them their own channel. Vocals therefore get a distinct 4th accent color, shown in the waveform as a **highlight overlay** on the sections where vocal energy is detected (rather than a blended channel). This keeps the core Low/Mid/High → R/G/B mapping honest while still giving vocals a consistent, learnable color.
- Shown on the track detail screen (scrubbable/zoomable) and as a scrolling backdrop during gameplay.

### 6.5 Light / Dark Mode
- Full theme support, follows system setting by default with a manual override in settings.
- Applies to library, track detail, gameplay, and results screens.

### 6.6 Gameplay Loop (falling-marker / lane-based style)
- **Lanes:** 4 fixed lanes at the bottom of the screen, one per marker type — **Low (Bass), Mid, High, Vocals** — using the **shared color system from 6.4** (Low=Red, Mid=Green, High=Blue, Vocals=accent). Each lane also has a distinct marker *shape* (not just color) so the coding survives color-blindness and both light/dark themes — e.g. Low=square, Mid=circle, High=triangle, Vocals=diamond (exact shapes finalized in the design pass).
- **Marker flow:** markers spawn at the top of the screen and fall toward a hit line above the lane buttons, timed so they reach the line exactly when their corresponding sound occurs in the track (i.e., the fall duration is a fixed lead time, e.g. 2 seconds, giving the player a consistent reaction window).
- **Input:** on-screen tap targets, one per lane, aligned under each lane. Player taps the correct lane's button as its marker crosses the hit line.
- **Note types:** MVP ships with **tap markers** (instantaneous hits). The marker data model, however, carries a `type`/`duration` field from the start so **hold/sustain notes** (tap-and-hold across a sustained bass or pad) and later **slide notes** can be added without reworking the schema — borrowed from Beatstar's note variety. Hold notes are a fast-follow, not MVP.
- **Timing judgment:** each hit is scored against how close (in ms) the tap was to the marker's ideal timestamp, **after subtracting the per-device latency offset from calibration (see 6.7).** The exact Perfect/Good/Bad windows **depend on the selected difficulty (see 6.8)** — e.g. on Normal: Perfect ±30ms, Good ±80ms, Bad ±150ms, Miss beyond that or no input before the marker passes.
- **Scoring:** Perfect = 100 pts, Good = 50 pts, Bad = 10 pts, Miss = 0 and breaks combo. Consecutive non-miss hits build a combo multiplier (e.g., +10% per 10-combo, capped) to reward sustained accuracy.
- **Session flow:** pick a track → short countdown → track plays back while markers fall → results screen at the end showing score, accuracy %, max combo, per-lane breakdown (helps a DJ see e.g. "I'm weak on catching vocal entries").
- **Playback control:** pause/resume; a track can be replayed anytime from the library.

### 6.7 Latency Calibration (MVP requirement)
- A one-time (re-runnable) **calibration screen** measures the combined audio-output + display + input latency of the specific device and stores a per-device offset in ms.
- Flow: the player taps along to a simple steady beat; the app computes the average offset between the beat and their taps and applies it to all future timing judgments.
- **Why it's in MVP, not optional:** device audio/display/touch latency varies widely and directly corrupts a timing-sensitive game — without calibration, "Perfect" hits register as "Good/Bad" on slower devices and the game feels broken. This is the single most-copied lesson from Cytoid. Prompted during onboarding and accessible anytime from Settings.

### 6.8 Difficulty Tiers (MVP)
Player picks **Easy / Normal / Hard** per play session (chosen after selecting a track). The difficulty affects two independent knobs, both served from the *same* cached analysis:

| Difficulty | Marker density | Timing windows (Perfect / Good / Bad) |
|---|---|---|
| **Easy** | Sparse — only the highest-confidence onsets surface (e.g. strong downbeats & obvious bass hits); fewer simultaneous markers | Forgiving — e.g. ±50 / ±120 / ±200 ms |
| **Normal** | Moderate — main rhythmic events across all four lanes | Standard — ±30 / ±80 / ±150 ms |
| **Hard** | Dense — most detected onsets surface, including subtler mid/high events and faster sequences | Tight — e.g. ±20 / ±50 / ±100 ms |

- **Density is controlled by a confidence threshold + a per-lane rate cap**, not by genre. The analysis pass stores every onset with a confidence score; each difficulty simply admits onsets above its threshold (with a cap on markers-per-second per lane so dense tracks stay playable). This means **marker density scales with difficulty, independent of musical genre** — a sparse ambient track and a dense DnB track both get an appropriate-feeling chart at each level.
- Scores are tracked per-difficulty (a Hard clear is a distinct record from an Easy clear on the same track).

## 7. Permissions & Privacy

- Folder/file access permission requested with a clear explanation of why (to read locally stored music for analysis).
- No audio file, waveform, or metadata is ever uploaded or transmitted off the device — all analysis, storage, and gameplay are 100% local. This is a hard requirement given the copyrighted nature of users' music libraries.
- No account/sign-in required for MVP.

## 8. Non-Functional Requirements

- **Minimum OS: iOS 16+ and Android 10 (API 29)+.** Rationale: (1) Android 10 is where **scoped storage / Storage Access Framework** stabilized, which our folder-access model depends on; (2) both cover the large majority of devices still in active use, and DJs — who invest in gear — skew toward newer hardware, so a modern floor costs us very little audience; (3) a newer floor gives the DSP pipeline more consistent performance to target. This is a starting decision — Phase 0 may raise the Android floor slightly if older mid-range chips can't keep up with the analysis workload.
- Analysis of an average 4-minute track should complete in well under real-time (target: under ~10s on a mid-range device) so importing a library doesn't feel like a blocker; show per-track progress in the library view.
- Gameplay must hold a steady 60fps for marker animation; any dropped frames directly hurt timing-sensitive input.
- App must work fully offline after initial install.
- Handle libraries of at least a few hundred tracks without noticeable UI lag (virtualized list, indexed local DB).

## 9. Key Technical Risks

| Risk | Detail | Mitigation |
|---|---|---|
| iOS folder access is limited | Apple's sandboxing means there's no true "grant a folder, watch it forever" like Android SAF; user must re-pick via document picker, and security-scoped bookmark access can be revoked by the OS | Design import as an explicit "Add folder" action the user repeats as needed; store bookmarks and gracefully prompt re-grant if access is lost |
| DSP performance/accuracy | Custom native FFT/onset-detection module is the riskiest build item — BPM/onset algorithms are nontrivial to get right, and performance must hold on lower-end devices | Spike this first (Phase 0) before committing to the rest of the build; consider a proven open-source DSP library (e.g. an Aubio-based native wrapper) rather than writing beat-tracking from scratch |
| Frequency-band heuristic ≠ real instrument detection | A "Vocals" marker is really "mid-band harmonic energy," which will sometimes fire on synths/leads, not just voice; players may find markers occasionally feel "wrong" | Set expectations in-app copy (e.g. "Melody/Mid" instead of strictly "Vocals" if needed); tune thresholds per genre if time allows; treat as a known MVP limitation |
| File format/codec support gaps | FLAC/OGG decode support isn't uniform out of the box on both OSes | Confirm decode path per format in Phase 0 spike; may need to trim supported formats for MVP if a codec proves unreliable |
| DRM-protected files | Apple Music downloads, protected files, etc. can't be decoded | Detect and clearly message "unsupported/protected file" rather than failing silently |

## 10. Phased Roadmap

- **Phase 0 — Technical spike:** Validate folder access on both platforms and validate the native audio decode → FFT → BPM/onset pipeline end-to-end on one sample track each platform. This phase de-risks the whole project before UI work starts.
- **Phase 1 — Library foundation:** Folder import, file scanning, metadata/artwork extraction, library list UI, light/dark theme shell.
- **Phase 2 — Analysis pipeline:** Full BPM detection, waveform + RGB band extraction, onset/marker generation, local caching, per-track analysis progress UI.
- **Phase 3 — Core gameplay:** Lane rendering, falling markers, hit-line input handling, **latency calibration screen (6.7)**, **Easy/Normal/Hard difficulty (6.8)**, timing judgment, scoring, results screen.
- **Phase 4 — Polish:** Animations/juice, RGB waveform backdrop during play, settings screen (incl. re-run calibration), onboarding/permission flow, empty states.
- **Phase 5 — Post-MVP (not in initial build):** Hold/sustain & slide note types, audio-based key detection (chroma analysis) for untagged tracks, loop/practice mode, slow-down practice speed, per-track cue-point notes, additional marker types.

## 11. Success Metrics (informal, since this is a personal/indie project)

- A DJ can import their real library and get usable BPM/key/artwork for the large majority of tracks without manual fixes.
- Markers "feel" aligned with what a listener actually hears (subjective playtesting pass against a handful of known tracks).
- A full play session (import → analyze → play → see results) works end-to-end offline on both a real iOS and real Android device.

### Resolved decisions
- **BPM:** Kept for MVP (detect from audio + read tag when present). Low risk.
- **Key:** Read-from-tag only for MVP; show "Unknown" when untagged. Audio-based key detection deferred to Phase 5.
- **Latency calibration:** Promoted into MVP (Phase 3) as a requirement.
- **Note types:** MVP = tap only, but marker schema supports hold/slide for a post-MVP fast-follow.
- **Difficulty:** Easy/Normal/Hard in MVP (see 6.8).
- **Marker density:** Scales with **difficulty**, not musical genre — via a confidence threshold + per-lane rate cap on a single cached analysis (see 6.8).
- **Color system:** Marker/lane colors and RGB waveform bands share one palette (Low=R, Mid=G, High=B, Vocals=accent) so the color code reinforces across both surfaces (see 6.4).
- **Minimum OS floor:** **iOS 16+ and Android 10 / API 29+** (rationale in §8). Revisit if Phase 0 shows the DSP struggles on the low end of that range.
- **Legal (gameplay style):** Falling-marker/lane rhythm mechanic is a non-copyrightable game system and free to use; we avoid the "Guitar Hero" trademark and use our own art/name.

## 12. Open Questions

1. **Naming:** working title is "DJ Rhythm Trainer." No name chosen yet — see the brainstorm list below; open to picking one or generating more.

### Name brainstorm (working candidates)
- **BeatVision** — nods to the RGB visual/waveform angle
- **CueDrop** — "cue" (DJ term) + falling markers
- **MixTrainer** — plainly says what it does
- **WaveRunner** — the scrolling RGB waveform + rhythm run *(note: check trademark — shared with a jet-ski brand)*
- **DropCatch** — catching the drop / catching markers
- **SpectraTap** — spectrum colors + tapping
- **FlowState** — the "flow" of a track the user described *(common phrase — check store collisions)*
- **KickCue** — kicks/beats + DJ cue

---

*Status: Draft, decisions locked in per §11 Resolved Decisions. Nothing has been built yet. Next step is Phase 0 (the technical spike) before any UI work, since it validates the riskiest assumptions first.*
