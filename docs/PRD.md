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
- BPM (detected, or tag value if embedded and trusted).
- Musical key, if present in tags (e.g., Camelot notation or standard key from `TKEY`/`initialkey`/Mixed In Key tags). Not computed from audio in MVP if untagged — see Open Questions.
- Duration, sample rate/bitrate (secondary info, e.g. an "info" expand panel).

### 6.3 Audio Analysis Pipeline (runs once per track, cached)
1. **Decode** the file to PCM via the native module.
2. **Waveform extraction:** downsample amplitude peaks per time window, split into low/mid/high frequency energy per window (via FFT) to drive the RGB waveform (see 6.4).
3. **BPM detection:** onset-strength/autocorrelation-based tempo estimation (standard beat-tracking DSP approach).
4. **Onset/band detection (marker generation):** per analysis window, run FFT and bucket energy into bands used as proxies for:
   - **Bass** — low-frequency band onsets (~20–250 Hz)
   - **Drums/Beats** — broadband transient/percussive onsets (spectral flux spikes)
   - **Vocals** — mid-frequency band with vocal-formant characteristics (~300 Hz–3.5 kHz, harmonic content)
   Each detected onset above a confidence threshold becomes a **marker** with a timestamp, lane/type, and intensity (used for point value / marker size).
5. Results (waveform data, BPM, key if tagged, marker timeline) are written to local storage as a compact JSON/binary blob per track.

This is a **precompute-once, play-from-cache** design — no real-time DSP during gameplay, which keeps the game loop simple and battery-friendly.

### 6.4 RGB Waveform Display
- Full-track waveform rendered with color channels mapped to frequency bands (e.g., R = bass energy, G = mid/vocal energy, B = high/percussive energy) per time slice, so a glance at the waveform shows where the track is heavy in bass vs. vocals vs. highs.
- Shown on the track detail screen (scrubbable/zoomable) and as a scrolling backdrop during gameplay.

### 6.5 Light / Dark Mode
- Full theme support, follows system setting by default with a manual override in settings.
- Applies to library, track detail, gameplay, and results screens.

### 6.6 Gameplay Loop (Guitar Hero–style)
- **Lanes:** 4 fixed lanes at the bottom of the screen, one per marker type — Bass, Drums, Beats, Vocals — each with a distinct color and marker shape (e.g., Bass = red square, Drums = orange circle, Beats = yellow diamond, Vocals = blue triangle — exact palette TBD in design pass, must hold up in both light and dark mode).
- **Marker flow:** markers spawn at the top of the screen and fall toward a hit line above the lane buttons, timed so they reach the line exactly when their corresponding sound occurs in the track (i.e., the fall duration is a fixed lead time, e.g. 2 seconds, giving the player a consistent reaction window).
- **Input:** on-screen tap targets, one per lane, aligned under each lane. Player taps the correct lane's button as its marker crosses the hit line.
- **Timing judgment:** each hit is scored against how close (in ms) the tap was to the marker's ideal timestamp:
  - **Perfect:** within ±30ms
  - **Good:** within ±80ms
  - **Bad:** within ±150ms
  - **Miss:** outside ±150ms or no input before the marker passes
  (exact windows tunable; see Open Questions on difficulty tiers)
- **Scoring:** Perfect = 100 pts, Good = 50 pts, Bad = 10 pts, Miss = 0 and breaks combo. Consecutive non-miss hits build a combo multiplier (e.g., +10% per 10-combo, capped) to reward sustained accuracy.
- **Session flow:** pick a track → short countdown → track plays back while markers fall → results screen at the end showing score, accuracy %, max combo, per-lane breakdown (helps a DJ see e.g. "I'm weak on catching vocal entries").
- **Playback control:** pause/resume; a track can be replayed anytime from the library.

## 7. Permissions & Privacy

- Folder/file access permission requested with a clear explanation of why (to read locally stored music for analysis).
- No audio file, waveform, or metadata is ever uploaded or transmitted off the device — all analysis, storage, and gameplay are 100% local. This is a hard requirement given the copyrighted nature of users' music libraries.
- No account/sign-in required for MVP.

## 8. Non-Functional Requirements

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
- **Phase 3 — Core gameplay:** Lane rendering, falling markers, hit-line input handling, timing judgment, scoring, results screen.
- **Phase 4 — Polish:** Animations/juice, RGB waveform backdrop during play, settings screen, onboarding/permission flow, empty states.
- **Phase 5 — Post-MVP (not in initial build):** Loop/practice mode, slow-down practice speed, per-track cue-point notes, difficulty tiers, additional marker types.

## 11. Success Metrics (informal, since this is a personal/indie project)

- A DJ can import their real library and get usable BPM/key/artwork for the large majority of tracks without manual fixes.
- Markers "feel" aligned with what a listener actually hears (subjective playtesting pass against a handful of known tracks).
- A full play session (import → analyze → play → see results) works end-to-end offline on both a real iOS and real Android device.

## 12. Open Questions

1. **Untagged key detection:** if a track has no embedded key tag, should MVP attempt audio-based key detection (chroma/pitch-class analysis), or just show "Unknown" and leave key detection as post-MVP? *(Leaning: show "Unknown" for MVP — audio key detection is a second nontrivial DSP feature on top of BPM/onsets.)*
2. **Difficulty tiers:** one fixed timing-window difficulty, or Easy/Normal/Hard with different marker density and timing windows?
3. **Marker density/genre handling:** should marker density scale with track complexity (e.g., fewer markers for sparse ambient tracks, more for dense drum & bass), or a fixed target markers-per-minute regardless of genre?
4. **Device support floor:** any minimum OS version / device age to target, given the DSP workload (affects how conservative the analysis pipeline needs to be)?
5. **Naming:** working title is "DJ Rhythm Trainer" — do you have an actual app name in mind?

---

*Status: Draft for review. Nothing has been built yet — once you sign off (or send edits), the plan is to start with Phase 0 (the technical spike) before any UI work, since it validates the riskiest assumptions first.*
