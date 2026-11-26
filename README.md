# Camera Face Capture POC

A single static page (`index.html`). It opens the device camera (with a picker if there are several), finds faces in the live feed, saves one snapshot for each unique person and logs every time that person is seen. Everything runs in the browser. Nothing is uploaded.

**Live demo:** https://cheokjiade.github.io/video-face-detection/

## How it works

1. While the camera runs, the page copies the current frame onto a canvas every *N* seconds (0.5 s by default).
2. It runs three models from [`@vladmandic/face-api`](https://github.com/vladmandic/face-api) on that frame:
   - SSD MobileNetV1 finds the faces.
   - A 68-point landmark model lines each face up.
   - A ResNet model turns each face into a 128-number descriptor.
3. It compares each descriptor with the faces already found, using Euclidean distance. A distance below the threshold (0.55 by default) means the same person, and the time is added to that person's list. Otherwise it saves a cropped snapshot as a new unique face.
4. Each log line shows how long the scan took (`[NN ms]`), so you can check whether a device keeps up.

To run it locally, run `npx serve` and open the localhost URL. The camera needs HTTPS or localhost, so opening the file directly may not work. The models load from the jsDelivr CDN.

## Hardware requirements

> **These numbers are estimates.** They come from the published model sizes and typical TensorFlow.js WebGL performance. They have **not** been measured on real devices. To measure, open the page on the target device and read the `[NN ms]` value in the log.

**What "smooth" means here:** each scan has to finish faster than the scan interval. With the default 0.5 s interval, a scan time under about 300 ms leaves room for the camera preview to stay smooth.

**Cost of one scan:** one face detection on the frame, plus landmarks and a descriptor for *each* face. More faces in the frame means a longer scan.

| Device (released in the last 2 years) | Expected scan time (1–3 faces) | Verdict at 0.5 s interval |
|---|---|---|
| Laptop, Apple M-series or Intel Core Ultra / AMD Ryzen 7000+ with integrated GPU | ~30–100 ms | Smooth. Can scan every 0.1–0.2 s |
| Laptop with a discrete GPU | ~15–50 ms | Smooth |
| Flagship phone (iPhone 15/16, Pixel 8/9, Galaxy S24/S25) | ~80–250 ms | Smooth |
| Mid-range phone (e.g. Galaxy A5x, Pixel 8a) | ~200–500 ms | Usable. Set the interval to 1 s |
| Budget phone (under ~$200) | 500 ms–1.5 s+ | Choppy. Use a 1–2 s interval, or move detection to a server |

**Minimum requirements:**
- **Browser:** a current version of Chrome, Edge, Safari or Firefox with WebGL2. The page shows the backend it is using in the status line (`webgl` is good; `cpu` will be very slow).
- **RAM:** about 300–500 MB for the browser tab (models, TensorFlow.js memory and video decoding). 4 GB of system RAM is enough. On phones, close other apps.
- **Download:** about 12 MB of model files on first load (detector 5.4 MB, descriptor model about 6 MB, landmarks 0.35 MB). After that the browser caches them.
- **Camera:** the page asks for 1280px wide. Every scan copies the full frame, so higher resolutions cost more memory and copy time and can stutter on phones.
- **Long sessions on phones:** the GPU running for a long time causes thermal throttling. Expect scans to get 1.5–2 times slower after a few minutes.

**When to move to a server:** if the target is low-end phones, or many faces per frame (crowds), run only face *detection* on the device (or none at all), and send the frames or crops to the planned API for the descriptor step.

## License report

> This is an engineering summary, not legal advice. Have counsel review it before any commercial or production use.

### Code and libraries

| Component | License | Notes |
|---|---|---|
| [`@vladmandic/face-api`](https://github.com/vladmandic/face-api) 1.7.15 | MIT | **The repo was archived in Feb 2025.** The author points to [`@vladmandic/human`](https://github.com/vladmandic/human) (MIT) as its successor. |
| [face-api.js](https://github.com/justadudewhohacks/face-api.js) (original project the fork is based on) | MIT | Keep the copyright notice if you redistribute it |
| TensorFlow.js (bundled inside face-api's `dist`) | Apache-2.0 | Keep the NOTICE/license if you redistribute it |
| This repository's own code | **No license chosen yet** | Without a LICENSE file, others may view the code but have no right to reuse it. Add MIT (or similar) if you want that. |

MIT and Apache-2.0 both allow commercial use. Their only real requirement is keeping the copyright and license notices.

### Model weights (the riskier part)

| Model | Source | License / status | Commercial risk |
|---|---|---|---|
| SSD MobileNetV1 face detector | Weights from [yeephycho/tensorflow-face-detection](https://github.com/yeephycho/tensorflow-face-detection), trained on WIDER FACE | Repo is Apache-2.0. WIDER FACE dataset terms are for **research use** | Medium. The code license is permissive, but whether dataset terms carry over to trained weights is legally unsettled |
| Face recognition (128-d descriptor) | Weights trained by Davis King (dlib) | [Public domain](https://github.com/davisking/dlib-models) ("anyone can do whatever they want"). Trained on FaceScrub, VGG-Face and scraped web images | Medium. The weights are free to use, but the training data includes scraped faces collected without consent |
| 68-point landmarks | Trained by the face-api.js author on "~35k face images", source not stated | MIT, the same as face-api.js | **Unclear.** The 68-point layout is the iBUG 300-W layout. dlib's own 68-point model is [explicitly *not* for commercial use](https://github.com/davisking/dlib-models) because of 300-W's license. If this model used 300-W data, the same restriction may apply |

**For commercial use, either:**
- check the landmark model's training data, or
- swap in a pipeline where every model's data is licensed for commercial use: a commercial SDK, or models trained on licensed or synthetic data.

### Laws on biometric data (more important than the software licenses)

Face descriptors count as **biometric data** in most places, even inside a POC once real people's footage is used:
- **EU/UK GDPR, Art. 9:** processing to identify people is a "special category". It needs explicit consent or another Art. 9 basis, plus a DPIA.
- **Illinois BIPA:** needs written consent and a retention policy. Individuals can sue for $1k–$5k per violation. Texas and Washington have similar laws.
- **Singapore PDPA:** consent and notification apply to facial images, and so does the guidance on biometric identification.
- **EU AI Act:** *real-time* remote biometric identification in public spaces is heavily restricted, and scraping faces to build recognition databases is prohibited.

**Why this POC is low-risk today:**
- Everything stays in the browser tab and nothing is stored or sent anywhere.
- It only groups faces *within one session*; it doesn't match them against a known identity database.

**This changes once snapshots or descriptors go to the planned API.** At that point you will need consent or notice, a retention limit, and encryption.
