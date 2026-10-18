# TBeat

Reads heart rate (BPM) from a phone camera with no contact, using remote photoplethysmography (rPPG). Everything runs in the browser and is built for iPhone Safari, served locally from iSH. There are four apps:

| File | What it is |
|---|---|
| `index.html` | **TBeat**: single-person BPM from the face (forehead and both cheeks). Checked by hand against an Apple Watch, and it matched closely. |
| `index-multi.html` | **TBeat Multi**: up to 6 people at once. Finds each person with coco-ssd, then their face with face-api, and tracks each one separately (TCount-style). |
| `index-multi-faceonly.html` | **TBeat Multi (faces only)**: up to 6 faces, using face-api alone. Lighter, but only works for faces close to the camera. |
| `index-grid.html` | **TBeat Radar**: fully offline, with no face detection. The picture is split into a grid of skin cells, and every cell is checked for a pulse. Cells beating at the same rate are grouped into one person, shown as a dot with their BPM on the video and on a radar display. It counts people like TCount: *In Frame* and *Unique*, with Stop & Total and Count Again. |

> ⚠️ **Not a medical device.** TBeat is an experiment for fun and learning. Don't use it to diagnose, monitor or treat anything. Readings can be wrong, especially with movement, poor or flickering light, or people far from the camera.

## Run on iPhone (iSH)
1. Copy the repo folder into iSH, for example `~/tbeat`.
2. Run `cd ~/tbeat && python3 -m http.server 8080`.
3. In Safari, open `http://127.0.0.1:8080/index.html`, or `index-multi.html`, `index-multi-faceonly.html` or `index-grid.html`.
   The camera needs `https://` or `127.0.0.1`. Opening the file straight from the Files app won't work.
4. Allow camera access, tap **Start** and hold still. A reading appears after about 10 s (the radar needs about 6 s per pulse).

The page needs internet access the first time: `index.html` and the two Multi versions load face-api.js and coco-ssd from jsDelivr. `index-grid.html` has no outside dependencies at all.

On a desktop, run the same server and open it in Chrome or Safari.

## Repo layout
| Path | Role |
|---|---|
| `index*.html` | The four apps. Each one is a self-contained file. |
| `gridsrc/head.html`, `gridsrc/app.js` | Source for `index-grid.html` = `head.html` + `<script>` + `app.js` + `</script></body></html>` |
| `legacy/src/` | Older split sources for the Multi version (`head.html`, `app.js`, `dsp.html`). Out of date, kept only for reference. |
| `test/` | Playwright and ffmpeg test scripts that build synthetic pulsing videos and run the apps on them (see `docs/HOW-IT-WORKS.md`) |
| `docs/HOW-IT-WORKS.md` | Detailed technical write-up |

## Limitations
- You need steady, even light, and people need to be close and mostly still. Head motion and light jumps lower confidence, and readings are hidden when confidence is "Low".
- Radar mode needs skin patches of at least about 3 cells. Faces have to be about 110 px tall at the default grid, or about 55 px at Grid 64.
- Radar mode merges two people side by side who have the same BPM into one. A pulse close to a flickering light's frequency can be masked out.
- It hasn't been checked against clinical equipment.

## License and credits
- Recommended license for the original code: **MIT**. No LICENSE file has been added yet.
- POS algorithm: W. Wang, A. C. den Brinker, S. Stuijk, G. de Haan, "Algorithmic Principles of Remote PPG," *IEEE TBME* 2017. This is an original re-implementation.
- Loaded at runtime (not bundled): [@vladmandic/face-api](https://github.com/vladmandic/face-api) 1.7.15 (MIT), [TensorFlow.js](https://github.com/tensorflow/tfjs) and [coco-ssd](https://github.com/tensorflow/tfjs-models) 2.2.3 (Apache-2.0).

Author: Jesus Teran ([@phymn](https://github.com/phymn))
