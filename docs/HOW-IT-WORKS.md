# TBeat: How it works

## 1. Principle
Each heartbeat pushes a little more blood into the skin, which changes how much green and red light the skin absorbs by a tiny amount (about 0.1–1 %). A camera can pick up that change in the average skin colour of a region over time. TBeat turns this colour signal into a pulse wave with the **POS** (Plane-Orthogonal-to-Skin) method, then finds the strongest frequency between **0.7 and 3.0–3.5 Hz**, which is 42–180/210 BPM.

All four apps share one DSP chain:

```
camera frame ─► ROI / cell mean RGB (skin pixels) ─► timestamped samples
  ─► resample to uniform FS (linear interpolation)
  ─► POS (1.6 s sliding window, overlap-add)
  ─► linear detrend ─► band-pass 0.7–3.0/3.5 Hz (2nd-order Butterworth HP + LP, forward-backward = zero phase)
  ─► Hann window, zero-padded radix-2 FFT (≥4× length, or NFFT=512 in Radar)
  ─► peak bin in band + parabolic interpolation ─► BPM
  ─► SNR = power within ±0.1 Hz of the peak and its 2nd harmonic ÷ the rest of the band (dB)
```

### DSP functions (the `<script id="dsp">` block in index.html and Multi; `gridsrc/app.js` in Radar)
| Function | Role |
|---|---|
| `resample(s, fs)` | Turns samples `{t (ms), r, g, b}` with uneven timing into evenly spaced R/G/B arrays at `FS`, using linear interpolation. Browsers deliver frames irregularly, so this step is needed. |
| `pos(R,G,B,fs)` | POS: in each window of `L = 1.6·fs` samples, divides by the window mean, projects onto `S1 = G−B` and `S2 = −2R+G+B`, and combines them as `h = S1 + (σ1/σ2)·S2`. The window is mean-centred and overlap-added into `H`. This cancels motion and specular changes that affect all three channels equally. |
| `detrend(x)` | Least-squares removal of a straight-line trend. |
| `biquad`, `bandpass(x, fs, lo, hi)` | 2nd-order Butterworth high-pass then low-pass (bilinear transform, Q = 1/√2), run forward and then backward so there's no phase lag. |
| `fftMag(x)` | Hann window, zero padding to the next power of two ≥ 4N, iterative radix-2 FFT, power spectrum. |
| `estimate(samples, fs)` | The whole chain above. Returns `{bpm, snr, signal}`, or `null` when there's less than 5 s of data. In Multi the check is the time span, not the sample count, so it still works when the frame rate is low. |

### Skin pixel test
A pixel counts as skin when, in YCbCr, `77 ≤ Cb ≤ 127` and `133 ≤ Cr ≤ 173` (Radar also requires `R > 40`). A region is only used when at least 20 % of its pixels are skin (10 % for small faces in Multi).

## 2. `index.html` (single person)
- **Camera**: `getUserMedia`, front camera by default (the ⟲ button switches cameras), 1280×720 @ 30 fps ideal. Frames are drawn to an offscreen canvas no more than 320 px wide.
- **Face**: face-api `TinyFaceDetector` (inputSize 320, score ≥ 0.4) runs every 333 ms on the WebGL backend. Models load from jsDelivr.
- **ROIs** (`roiList`): forehead (30–70 % of the face width, 8–23 % of its height) and two cheeks (15–35 % and 65–85 % of the width, 50–68 % of the height). `avg()` returns the skin-pixel mean RGB of each.
- **Samples**: the combined mean of all valid ROIs goes into `buf` (20 s kept), and each ROI also keeps its own buffer.
- **Motion and light**: a face move of more than 5 % of the face width counts as a move. A brightness jump of more than 8/255 counts as a light jump.
- **`compute()`, once per second over the last 10 s**: estimates from the combined signal and from each ROI. `agree` is how many ROIs land within 8 BPM of the combined estimate. Score = (SNR > 0 → 2, > −3 → 1) + (agree ≥ 2 → 1) − (moves > 3 → 1) − (jumps > 2 → 1). Confidence is **High** at ≥ 3, **Med** at ≥ 2, and **Low** otherwise. The shown BPM is the median of the last 5 estimates and is hidden when confidence is Low or before 10 s.
- **UI**: BPM, confidence, progress bar, live filtered waveform, warnings ("No face found", "Too far"), and CSV export (timestamp, BPM, confidence, SNR).

## 3. `index-multi.html` (up to 6 people)
- **Detection cycle** (`detectCycle`, every `DET_MS` = 300 ms): the zoomed view is copied to a canvas no more than 640 px wide.
  - *Person mode* (the default once coco-ssd loads): coco-ssd `detect(…, 20, 0.3)` → persons with score > 0.4, at most 6. For each tracked person, `faceInPerson()` crops the head area (the top 35 % of the body box, square, ×1.1) from the **full-resolution** frame, scales it up to 160×160 and runs face-api (score ≥ 0.35). This is the trick TCount uses for small faces. Up to 3 people are refreshed per cycle, least recently tried first, and between refreshes the face box moves along with the body box. Faces that are still missing fall back to a full-frame face search matched to each body's head position. If no persons are found at all, it uses faces only.
  - *Face mode*: face-api `detectAllFaces` (score ≥ 0.45).
  - **Adaptive detector size** (`autoSize` ∈ {224, 320, 416, 512, 608}): steps up when the smallest face is under 40 px, or nothing is found, and detection takes under 450 ms. Steps down when faces are over 110 px or detection takes over 700 ms.
- **Tracking** (`associate`, the same rules as TCount): pair score = 2·IoU − centroid distance / frame diagonal, greedy matching, minimum 0.15. Unmatched boxes that overlap an existing track by IoU > 0.3 are treated as duplicates. Box movement is smoothed with `lerpBox` (α = 0.35, or a jump when it moves more than 25 %). Tracks are marked missed after `MISS_MS` = 1.5 s and dropped after `DROP_MS` = 4 s. Faces are kept for `FACE_HOLD` = 5 s.
- **Sampling** (`grabFaces`, `sampleTrack`): one `getImageData` call per frame covers the bounding rectangle of all faces, at full resolution and no more than 720 px wide. Larger ROIs are used (forehead 25–75 % × 5–25 %, cheeks 12–38 % and 62–88 % × 48–72 %). ROI means are weighted by their pixel count. Each track flags `small` (face < 40 px), `dark` (brightness < 55) or `bright` (a channel > 240).
- **Per-person `compute()`**: a 10 s window, or 15 s for small or weak signals, with band 0.7–3.5 Hz. Quality score 0–100 = SNR term (up to 55) + ROI agreement (up to 25) + stability of the last 5 estimates (up to 20 − spread) − motion 20 − light jumps 15 − small 10 − dark 10. **High** ≥ 65, **Med** ≥ 40. BPM display = EMA (α = 0.4) of the median of 5. A person is marked ready after `MEASURE_MS` = 10 s. History is kept for sparklines.
- **UI**: coloured boxes with ID and BPM, a list with quality, global warnings, a zoom slider (native camera zoom when Safari exposes it, otherwise a CSS crop), a debug panel, and CSV export (time, ID, BPM, quality).

## 4. `index-multi-faceonly.html`
The same per-person pipeline as Multi, minus coco-ssd: face-api `detectAllFaces` for up to 6 faces (`MAX_FACES`), the same IoU/centroid association, ROIs and quality scoring. It's lighter on the phone, but faces have to be large enough for the face detector.

## 5. `index-grid.html`, TBeat Radar (offline, no detection)
Source: `gridsrc/head.html` (UI/CSS) + `gridsrc/app.js` (logic).

### Constants
`N = 4` pixels per cell side in the sampled image, `FS = 15` Hz resample rate, `NFFT = 512`, band `LO = 0.7`–`HI = 3.5` Hz, `MAXP = 6` people shown, `CAP_MS = 29`. Spectrum resolution `DF = FS/NFFT ≈ 0.029 Hz`, and the peak width is ±0.1 Hz.

### Grid and sampling
- `setupGrid()`: the long side has `L` cells (Auto starts at 32, then 48, then 64; or a fixed 16/24/32/48/64), and the short side scales by aspect ratio, with at least 4. A ring buffer holds about 15.5 s of frames: `T` timestamps, `D` the per-cell mean R, G, B and skin share, `BG` the background mean colour.
- `grab()`: a canvas **pyramid** halves the image until it's close to `cols·4 × rows·4`, then does one high-quality draw and **one `getImageData`** per frame. That gives a true area average and keeps it fast on iPhone.
- `sample()`: for each cell it stores the whole-cell mean RGB (stable) plus the share of skin pixels. The skin share is only used to *select* cells, because per-pixel skin decisions flicker from frame to frame. Non-skin, non-clipped cells feed a per-frame **background colour** used for light compensation.

### `analyze()`, once per second, over the window (10, 12 by default, or 15 s)
1. **Flicker detection**: up to about 120 background cells (skin share ≤ 2 %) are normalised (G+R), detrended, band-passed and run through the FFT. Their spectra are summed. Any local peak more than 6× the median is a **light flicker** frequency. That bin, and its 2nd harmonic, are masked (± width) for every cell.
2. **Per-cell POS**: cells with skin share ≥ 25 % are candidates. If "light compensation" is on and enough background cells exist (≥ 4 and ≥ 5 % of cells), each cell's RGB is divided by the background RGB. That cancels global light changes such as auto-exposure and flicker. A cell is marked **motion** (skipped) when its brightness jumps more than 4 % more than 4 times, or its skin share swings more than 0.7. Otherwise: POS → detrend → band-pass → normalised spectrum → masked peak, which gives per-cell BPM and SNR. An SNR-weighted sum of all cell spectra gives a "room" BPM.
3. **Clustering into people**: candidates with SNR > −5 dB are seeded from highest SNR down. **Region growing** spreads to the 8 neighbours whose BPM is within ±4 of the seed. Each region's spectra are summed with weights 10^(SNR/10) → the region's peak BPM, SNR, weighted centre and bounding box. Regions are then **merged** when their BPMs are within 4 and their centres are within max(3, 12 % of cols) cells. This joins one person's face and hand, or a forehead split by hair. A region is kept when it has ≥ 3 cells and SNR > −1, or 2 cells and SNR > 1.5, or 1 cell and SNR > 3. Quality = SNR term (up to 60) + size (4 per cell, up to 25) + agreement between cells (15 − 3·SD of the cell BPMs). The top 6 are kept, ranked by `snr + 3·ln(1+cells)`.
4. **Auto grid**: if no source has been found for 20 s, the grid steps up to 48 and then 64, for people farther away.

### Counting (the `tracker` object, TCount rules plus pulse checks)
- Match score = 2·IoU − centroid distance / diagonal, minus 1 when the BPMs differ by more than 8. Greedy matching, ≥ 0.15.
- **Pulse pass**: an unmatched source within 6 BPM and less than 0.15 of the diagonal from an unmatched track is the same person, even if the cell shape changed.
- Only detections with quality ≥ 45 add hits. A track is **confirmed and counted once** at `minHits` = 6 hits, which is about 6 s with one analysis per second. Tracks die after `maxAge` = 5 missed seconds.
- **Re-acquire**: confirmed tracks that vanished go to a `lost` list for 15 s. A new source within 6 BPM and less than 0.12 of the diagonal of a lost one gets the old ID back and **isn't counted again**.
- Counters: **In Frame** = confirmed tracks updated this second. **Unique** = confirmed IDs this session. **Stop & Total** shows the final unique count, and **Count Again** resets everything.

### UI
Coloured dots and boxes with BPM on the video, a heat-map toggle showing per-cell SNR, a **radar** canvas (a sweep line every 3 s, with dots placed at each source's grid position), a list per person (ID, BPM, quality, cell count), room BPM, warnings (motion, flicker found, no skin, too dark), a debug panel, front/back camera switch, 1080p/720p, zoom, and CSV export (`tbeat-radar-<time>.csv`).

## 6. Settings, and why these values
- A 1.6 s POS window holds at least one heartbeat at 42 BPM, which is the value from the POS paper.
- A 10–15 s window gives the FFT 0.07–0.1 Hz (4–6 BPM) of raw resolution. Zero padding plus parabolic interpolation refines it further. Longer windows are steadier but slower to react.
- The 0.7 Hz lower band edge rejects breathing at 0.1–0.5 Hz. The 3.0–3.5 Hz upper edge covers heart rates up to exercise levels.
- An 8 BPM agreement and 6 BPM re-ID tolerance allow for estimation noise of a few BPM, while still telling apart people whose rates differ.
- minHits 6 / maxAge 5 (Radar) work out to about 6 s to confirm and 5 s to forget, which suppresses one-second false pulses.

## 7. Testing done (`test/`)
- `gen.py`, `gen2.py`, `gen3.py`, `gen4.py` build synthetic **Y4M** videos with ffmpeg: a background photo (`s1.jpg`–`s3.jpg`, not in the repo, so use your own) with skin regions or faces (`face_left.png`, `face_mid.png`, not in the repo) whose colour is modulated at known frequencies of 60–129 BPM. They can add noise, coloured light flicker (`gen4.py flickerAmp freq`) and several face sizes (14–220 px).
- `run.py`, `run2.py`, `run3.py` (Multi) and `rgrid.py` (Radar) start Chromium through Playwright with `--use-file-for-fake-video-capture`, run the page for N seconds, and compare the detected BPM, IDs and positions with the ground truth. They use environment variables for mode, face size, zoom, grid, window and compensation. `gflow.py` checks the Radar UI flow (CSV, camera switch, Stop & Total). `wk.py` takes a screenshot of the iPhone viewport in WebKit. `dist.sh` and `dist2.sh` run distance and zoom sweeps. `celldump.py` and `err.py` are debugging helpers.
- Results: in simulation, Radar measured every pulse it detected within 0.5 BPM. Flicker was rejected, and there were no double or false counts. The detection limit is a face about 110 px tall at Grid 32, or about 55 px at Grid 64. Single-person `index.html` was checked by hand against an Apple Watch. Real-camera testing of Radar and Multi is still to do.
- Running the tests needs `pip install playwright numpy pillow`, `playwright install chromium`, ffmpeg, and `python3 -m http.server 8080` running in the repo root.

## 8. Known limits and future work
- Two people side by side with the same BPM merge into one in Radar. Adding phase or waveform shape to the clustering could separate them.
- A pulse close to a flicker frequency gets masked.
- Radar can't tell who someone is: a person who leaves for more than 15 s, or comes back somewhere else, is counted again.
- The Multi versions need internet the first time, for the CDN models. They could be bundled for offline use the way TCount crowd is.
- Possible additions: HRV and breathing rate, a signal-quality ROI optimiser, a Web Worker for the per-cell FFTs, and bundling face-api offline.
- Not a medical device.
