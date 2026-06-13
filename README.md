# FFT Frequency Analyzer — Guitar String Tuning

A browser-based frequency analyzer built for an IB HL Physics Internal Assessment investigating the relationship between string gauge, tension, and fundamental frequency in guitar strings.

## Overview

This is a single-file HTML application that uses your device's microphone and the Web Audio API to measure the fundamental frequency of a plucked guitar string. It performs real-time FFT analysis and autocorrelation to detect pitch with high precision, runs automated multi-trial sequences, and exports data to CSV for further analysis.

Unlike generic spectrum analyzer apps, this tool is purpose-built for the physics lab: it filters readings by expected frequency range, enforces confidence thresholds, automates trial collection, and computes pooled statistics per string gauge.

## Quick Start

1. **Open the file** — Open [`FFT_Frequency_Analyzer.html`](FFT_Frequency_Analyzer.html) in any modern browser (Chrome, Edge, or Firefox recommended).

2. **Allow microphone access** — When prompted, grant permission for the browser to use your microphone.

   > **Note:** If microphone access is blocked when opening via `file://`, serve the file over a local server:
   > ```bash
   > python3 -m http.server 8000
   > # then open http://localhost:8000/FFT_Frequency_Analyzer.html
   > ```

3. **Select your string gauge** — Choose from the preset gauges (.009–.042) or edit the list to match your strings.

4. **Initialize the microphone** — Click **Initialize Microphone** and pluck a string to verify the frequency reading.

5. **Run a trial sequence** — Click **Run 5-Trial Sequence** to automatically record five plucks. Each trial captures ~3 seconds of sustain after detecting a pluck onset.

6. **Export your data** — Click **Export CSV** to download a spreadsheet with per-trial statistics and detailed frequency logs.

## How It Works

### Frequency Detection

The analyzer uses a two-stage pitch detection algorithm:

1. **Harmonic Product Spectrum (HPS)** — A coarse frequency estimate is found by downsampling the FFT spectrum and multiplying harmonics to reinforce the fundamental.
2. **Time-domain autocorrelation** — The coarse estimate is refined by searching autocorrelation lags near the HPS candidate, with parabolic interpolation for sub-sample precision.

### Signal Chain

```
Microphone → AnalyserNode (FFT size 32768) → HPS coarse detection
                                            → Autocorrelation refinement
                                            → Parabolic interpolation
                                            → Final frequency ± confidence score
```

### Trial Automation

Each trial:
- Waits for an RMS spike above the pluck detection threshold
- Records frequency readings for ~3 seconds of sustain
- Filters readings by confidence threshold and expected frequency range (±25%)
- Requires at least 20 valid readings to accept the trial

After 5 trials, statistics are computed and the interface prompts you to advance to the next string gauge.

## String Gauge Presets

| Gauge | Note | Expected Frequency |
|-------|------|--------------------|
| .009  | E4   | 329.63 Hz          |
| .011  | B3   | 246.94 Hz          |
| .016  | G3   | 196.00 Hz          |
| .024w | D3   | 146.83 Hz          |
| .032  | A2   | 110.00 Hz          |
| .042  | E2   | 82.41 Hz           |

You can edit gauges, target notes, expected frequencies, and linear densities via the **Edit Gauges** button.

## Exported Data

The CSV export includes two sections:

1. **Summary statistics** per gauge — mean frequency, standard deviation, min, max, range, and pooled statistics across trials.
2. **Detailed trial log** — every individual frequency reading with timestamp, trial number, and gauge.

## Technical Details

- **FFT size:** 32768 samples
- **Frequency range:** ~50–500 Hz (search window)
- **Sample rate:** Determined by the device's audio hardware (typically 44.1 kHz or 48 kHz)
- **Browser APIs:** Web Audio API, Media Capture API, Canvas 2D, Web Storage (localStorage)
- **Dependencies:** None (except Google Fonts — Outfit & JetBrains Mono — loaded from CDN)

Data is persisted in your browser's `localStorage` and remains available until you clear it or click **Clear All Data**.

## Project Structure

```
tune app/
├── FFT_Frequency_Analyzer.html   # The complete application
└── README.md
```

## Context

This tool was built for an IB Higher Level Physics Internal Assessment. The investigation examines how string gauge (linear mass density) affects the fundamental frequency of a vibrating guitar string under constant tension, relating experimental measurements to the standing wave equation:

$$f = \frac{1}{2L}\sqrt{\frac{T}{\mu}}$$

where $f$ is frequency, $L$ is string length, $T$ is tension, and $\mu$ is linear mass density.

## Browser Compatibility

| Browser | Status |
|---------|--------|
| Chrome / Edge | ✅ Fully supported |
| Firefox | ✅ Fully supported |
| Safari | ✅ Supported (may require HTTPS/localhost for microphone) |
| Mobile browsers | ✅ Responsive layout with touch-friendly controls |
