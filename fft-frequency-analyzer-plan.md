# Implementation Plan: Custom FFT Frequency Analyzer for IB HL Physics IA

## Summary

Build a single-file, self-contained HTML web application that serves as a precision FFT frequency analyzer for the IB HL Physics IA experiment "Part C: Measuring the Dependent Variable (Fundamental Frequency)." The app replaces Spectroid with a purpose-built tool featuring automated trial logging, statistical computation (mean, standard deviation), CSV export, and a distinctive, production-grade UI designed for macOS. Runs entirely in-browser via the Web Audio API — no installation, no dependencies, no server.

---

## Current State Analysis

### Existing Assets
- **research_design_updated.docx** (`/workspace/School/`): Contains the full methodology including Part C procedure (C1–C7), instrument specifications (Table 2), and experimental design.
- **IB_Physics_String_Analysis_v2.xlsx** (`/workspace/School/`): Contains all string data, μ calculations, uncertainty analysis, and theoretical f₀ values for all 6 gauges.

### Methodology Reference (from docx Part C)
| Step | Requirement |
|------|-------------|
| C1 | Quiet environment; smartphone mic ~10 cm above 12th fret midpoint |
| C2 | Pluck at midpoint, ~1 cm displacement, clean release |
| C3 | FFT spectrum recording for 2–3 seconds of stable vibration |
| C4 | Identify f₀ as lowest, most prominent peak; record in Hz |
| C5 | 5 trials per string; compute mean and standard deviation |
| C6–C7 | Repeat for all 6 gauges (.009, .011, .016, .024w, .032, .042) |

### Instrument Specifications (from Table 2)
- Spectroid FFT App: ±1 Hz resolution
- Target: match or exceed this with custom software

### Target Platform
- MacBook Pro 2022 (Touch Bar), macOS
- Browsers: Safari and Chrome (both fully support Web Audio API, getUserMedia, AnalyserNode)
- Sample rate: 44100 Hz or 48000 Hz (browser-dependent)

---

## Proposed Changes

### File to Create

**`/workspace/School/FFT_Frequency_Analyzer.html`** — Single self-contained HTML file (~800–1200 lines) with embedded CSS and JavaScript. No external dependencies.

---

### Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    UI Layer (HTML/CSS)               │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │ Gauge     │  │ Live     │  │ Trial Log Table   │  │
│  │ Selector  │  │ Spectrum │  │ (mean, σ, n, …)  │  │
│  │ + Trial   │  │ Display  │  │                   │  │
│  │ Controls  │  │ (Canvas) │  │ [Export CSV]      │  │
│  └──────────┘  └──────────┘  └───────────────────┘  │
├─────────────────────────────────────────────────────┤
│                 Audio Engine (JS)                    │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │ Mic       │  │ FFT      │  │ Pitch Detection   │  │
│  │ Capture   │──│ Analyser │──│ (Autocorrelation) │  │
│  │ (getUser  │  │ Node     │  │ + HPS refinement  │  │
│  │  Media)   │  │ fft=32768│  │                   │  │
│  └──────────┘  └──────────┘  └───────────────────┘  │
├─────────────────────────────────────────────────────┤
│              Data Layer (JS Classes)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │ TrialLogger  │  │ StatsEngine  │  │ CSVExport  │  │
│  │ (per-trial   │  │ (mean, σ,    │  │ (full      │  │
│  │  readings)   │  │  min, max)   │  │  session)  │  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
└─────────────────────────────────────────────────────┘
```

---

### Phase 1: Audio Engine — Pitch Detection

#### 1.1 Microphone Capture
- Use `navigator.mediaDevices.getUserMedia({ audio: { echoCancellation: false, noiseSuppression: false, autoGainControl: false } })`
- Disable all browser audio processing to preserve raw spectrum
- Handle `AudioContext` suspended state (required by both Safari and Chrome autoplay policies)
- Must be triggered by user gesture (button click)

#### 1.2 FFT Analyser Setup
- `fftSize = 32768` → frequency bin width ≈ 1.35 Hz at 44100 Hz sample rate
- `smoothingTimeConstant = 0` → no time-domain smoothing for accurate readings
- Use `getFloatFrequencyData()` for dB-precision (not byte-quantized)
- Frequency range of interest: 60–400 Hz (covers all 6 strings with margin)

#### 1.3 Pitch Detection Algorithm — Two-Stage Hybrid

**Stage 1: FFT + Harmonic Product Spectrum (HPS) — Coarse Detection**
1. Extract frequency data array from AnalyserNode
2. Restrict to bins corresponding to 60–400 Hz
3. Apply HPS: downsample spectrum by factors 2, 3, 4; multiply element-wise
4. Find bin index of maximum in HPS result
5. Convert to frequency: `f_coarse = binIndex × sampleRate / fftSize`

**Stage 2: Autocorrelation — Fine Refinement**
1. Capture time-domain buffer via `getFloatTimeDomainData()` (2048 samples)
2. Compute autocorrelation for lags corresponding to 60–400 Hz
3. Find lag with maximum correlation
4. Apply parabolic interpolation around peak for sub-sample accuracy
5. `f_refined = sampleRate / bestLag`

**Fallback**: If autocorrelation confidence is low, use HPS result directly.

#### 1.4 Onset Detection
- Monitor time-domain amplitude
- Threshold crossing (e.g., 0.05 normalized amplitude) triggers recording start
- Recording continues for configurable duration (default: 3 seconds)
- Auto-stop when amplitude drops below threshold for sustained period

---

### Phase 2: Data Layer — Trial Logging & Statistics

#### 2.1 TrialLogger Class
```javascript
class TrialLogger {
  // Properties
  trials: Array<{gauge, readings: number[], timestamp, stats}>

  // Methods
  startTrial(gauge)        // Begin new trial for given string gauge
  recordReading(freqHz)    // Append frequency reading (called ~10/sec during sustain)
  endTrial()               // Compute stats, finalize trial
  computeStats(readings)   // Returns {n, mean, stdDev, min, max, range}
  getAllTrials()           // Returns all completed trials
  getTrialsByGauge(gauge)  // Filter trials by gauge
  clearTrials()            // Reset all data
}
```

#### 2.2 StatsEngine
- Mean: `Σx / n`
- Sample standard deviation: `√(Σ(x - x̄)² / (n - 1))`
- Min, max, range
- Display with appropriate significant figures (matching instrument precision: 0.1 Hz)

#### 2.3 CSV Export
- Full session export with columns: `gauge, trial_number, n_readings, mean_Hz, stdDev_Hz, min_Hz, max_Hz, range_Hz`
- Per-reading detailed export: `gauge, trial_number, reading_index, frequency_Hz`
- Download triggered via Blob + URL.createObjectURL

---

### Phase 3: UI — Frontend Design (Skill: frontend-design)

#### 3.1 Aesthetic Direction: **"Laboratory Precision"**
- **Tone**: Scientific instrument — dark mode, high contrast, precision-focused. Think oscilloscope meets modern DAW plugin. Clinical but warm.
- **Typography**: 
  - Display/monospace for frequency readouts: **"JetBrains Mono"** (system-installed or Google Fonts)
  - Body: **"IBM Plex Sans"** — clean, technical, highly legible
- **Color Palette** (CSS variables):
  - Background: `#0a0e14` (deep near-black)
  - Surface: `#131820` (card backgrounds)
  - Border: `#1e2733` (subtle separators)
  - Primary accent: `#00e5ff` (cyan — frequency readouts, active state)
  - Secondary accent: `#ff6e40` (warm orange — record button, alerts)
  - Success: `#00e676` (trial complete, stable reading)
  - Text primary: `#e0e6ed`
  - Text secondary: `#6b7c93`
- **Spatial**: Three-column layout on desktop (≥1200px), single-column stacked on smaller screens. Generous padding, clear visual hierarchy.

#### 3.2 Layout Structure

```
┌──────────────────────────────────────────────────────────┐
│  HEADER: Title + Subtitle (IB HL Physics IA context)     │
├──────────────┬───────────────────────┬───────────────────┤
│  CONTROL     │  LIVE SPECTRUM        │  TRIAL LOG        │
│  PANEL       │  (Canvas)             │  (Table)          │
│              │                       │                   │
│  Gauge: [▼]  │  ┌─────────────────┐  │  Gauge  N  Mean  │
│  .009 E4     │  │                 │  │  .009   5  329.6 │
│  .011 B3     │  │  FFT Waterfall  │  │  .011   5  247.0 │
│  .016 G3     │  │  + Peak Marker  │  │  ...             │
│  .024w D3    │  │                 │  │                   │
│  .032 A2     │  │  f₀ = 329.6 Hz  │  │  [Export CSV]    │
│  .042 E2     │  │  ◄──────────►   │  │  [Clear All]     │
│              │  └─────────────────┘  │                   │
│  [▶ Record]  │                       │                   │
│  [■ Stop]    │  Status: ● Recording  │                   │
│              │  Trial 3/5 for .009   │                   │
│  Trial: 3/5  │                       │                   │
│  Readings:42 │  Confidence: ████░ 92%│                   │
└──────────────┴───────────────────────┴───────────────────┘
```

#### 3.3 Key UI Components

1. **Gauge Selector**: Dropdown or radio group with all 6 strings, showing gauge + note name + theoretical f₀
2. **Live Frequency Display**: Large monospace readout showing current detected f₀, updated in real-time (~10 Hz)
3. **Spectrum Canvas**: 
   - Real-time FFT magnitude plot (frequency vs dB)
   - Highlighted region for 60–400 Hz
   - Detected f₀ marked with vertical line + label
   - Harmonic markers at 2f₀, 3f₀ shown as dimmer lines
   - Subtle grid lines for readability
4. **Trial Controls**:
   - "Start Trial" button (prominent, warm accent color)
   - Auto-stop after 3 seconds or manual "Stop" button
   - Trial counter (e.g., "Trial 3 of 5 for .009 E4")
   - Reading count during active trial
5. **Trial Log Table**:
   - Columns: Gauge, Trial #, N Readings, Mean (Hz), Std Dev (Hz), Min, Max
   - Per-gauge summary row with pooled statistics
   - Color-coded: green for completed, amber for in-progress
6. **Export Controls**: "Export CSV" button, "Clear All" with confirmation

#### 3.4 Motion & Micro-interactions
- Smooth canvas rendering at 60fps via `requestAnimationFrame`
- Pulse animation on frequency readout when new value detected
- Subtle glow on record button during active trial
- Staggered fade-in for trial log rows
- Smooth color transition on status indicator (idle → recording → complete)
- CSS-only transitions (no JS animation libraries)

#### 3.5 Responsive Design
- Desktop (≥1200px): Three-column layout as shown above
- Tablet (768–1199px): Two-column (controls + spectrum stacked, log below)
- Mobile (<768px): Single column, stacked vertically
- Touch-friendly button sizes (≥44px tap targets)

---

### Phase 4: Automation Features

#### 4.1 Auto-Trial Mode
- User selects gauge, clicks "Run 5 Trials"
- System auto-detects each pluck onset, records 3 seconds, stops, waits for next pluck
- Visual countdown between trials (3-second rest)
- Auto-advances to next gauge after 5 trials complete
- Optional: prompt user to confirm before advancing

#### 4.2 Pre-configured Experiment Session
- Load all 6 gauges with theoretical f₀ values from the Excel workbook
- Display expected vs measured f₀ with percentage difference
- Flag readings that deviate >5% from theoretical (possible tension error)

#### 4.3 Data Validation
- Reject readings where confidence < 60% (no clear fundamental detected)
- Warn if detected f₀ is outside expected range for selected gauge (±20%)
- Minimum 20 readings per trial for statistical validity

---

### Phase 5: Technical Implementation Details

#### 5.1 File Structure (single HTML file)
```
FFT_Frequency_Analyzer.html
├── <style> — All CSS (~300 lines)
│   ├── CSS custom properties (theme)
│   ├── Layout (CSS Grid)
│   ├── Component styles
│   ├── Animations
│   └── Responsive breakpoints
├── <body> — HTML structure (~150 lines)
│   ├── Header
│   ├── Control panel
│   ├── Spectrum canvas
│   └── Trial log section
└── <script> — All JavaScript (~500 lines)
    ├── AudioEngine class
    │   ├── init() — getUserMedia, AudioContext, AnalyserNode
    │   ├── start()/stop() — audio processing loop
    │   ├── detectPitch() — HPS + autocorrelation
    │   └── detectOnset() — amplitude threshold
    ├── TrialLogger class
    │   ├── startTrial()/endTrial()
    │   ├── recordReading()
    │   └── computeStats()
    ├── UIManager class
    │   ├── renderSpectrum() — Canvas 2D drawing
    │   ├── updateFrequencyDisplay()
    │   ├── updateTrialLog()
    │   └── handleExport()
    └── Main controller (event bindings, app state)
```

#### 5.2 Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| FFT size | 32768 | ~1.35 Hz bin width, best achievable with AnalyserNode |
| Pitch algorithm | Autocorrelation (primary) + HPS (fallback) | Autocorrelation gives sub-Hz accuracy; HPS as backup |
| Canvas rendering | 2D Canvas API (not WebGL) | Sufficient for spectrum plot; simpler; better compatibility |
| Data persistence | In-memory only (no localStorage) | Session-based; CSV export for permanent storage |
| Font loading | Google Fonts via `<link>` | JetBrains Mono + IBM Plex Sans; fallback to system monospace |
| No frameworks | Vanilla HTML/CSS/JS | Zero dependencies; works offline; easy to audit for IA |

#### 5.3 Browser Compatibility Handling
- Feature detection for `navigator.mediaDevices.getUserMedia`
- Graceful error message if microphone access denied
- `AudioContext` resume on user gesture (Safari/Chrome requirement)
- Fallback font stack if Google Fonts unavailable

---

## Assumptions & Decisions

1. **Single HTML file**: Chosen for simplicity, portability, and IA submission requirements. No build step, no server needed.
2. **Autocorrelation over pure FFT**: Provides sub-Hz accuracy needed to match/exceed Spectroid's ±1 Hz. The cwilso/PitchDetect project validates this approach.
3. **Dark theme**: Appropriate for a lab instrument; reduces screen glare during experiment; matches oscilloscope/analyzer aesthetic.
4. **No audio processing**: Disabling echo cancellation, noise suppression, and auto-gain control is critical — these would distort the frequency spectrum.
5. **3-second recording window**: Matches the methodology (C3: "2–3 seconds to obtain a stable reading").
6. **Confidence threshold at 60%**: Below this, the autocorrelation peak is ambiguous — likely noise or multiple strings vibrating.
7. **Google Fonts loaded via CDN**: The app requires internet for first load (font caching), but fonts are not critical to function — graceful fallback to system fonts.

---

## Verification Steps

### Functional Verification
1. Open `FFT_Frequency_Analyzer.html` in Safari on MacBook Pro
2. Grant microphone permission when prompted
3. Verify live spectrum display updates in real-time
4. Pluck a guitar string — verify f₀ is detected and displayed
5. Run 5 trials for one gauge — verify mean and std dev are computed
6. Export CSV — verify file downloads with correct data
7. Repeat in Chrome — verify identical behavior

### Accuracy Verification
1. Use a known-frequency tone generator (e.g., online sine wave at 220 Hz)
2. Verify detected frequency is within ±1 Hz of 220 Hz
3. Test across range: 82 Hz, 110 Hz, 147 Hz, 196 Hz, 247 Hz, 330 Hz
4. Compare readings against Spectroid for same pluck (if possible)

### Edge Cases
1. No microphone — graceful error message
2. Microphone permission denied — clear instructions to re-enable
3. Very quiet environment (no pluck detected) — onset detection doesn't false-trigger
4. Background noise — confidence score drops, reading flagged
5. Multiple strings vibrating — HPS should still identify dominant f₀

---

## Implementation Order

1. **Audio Engine** — Core pitch detection (most critical, highest risk)
2. **Canvas Spectrum Display** — Visual feedback for debugging
3. **TrialLogger + StatsEngine** — Data recording and computation
4. **UI Layout + Styling** — Full frontend design implementation
5. **Trial Controls + Automation** — Auto-trial mode, gauge sequencing
6. **CSV Export** — Data export functionality
7. **Polish** — Animations, responsive design, edge case handling
8. **Testing** — Verification against known frequencies
