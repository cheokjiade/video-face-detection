# Camera Face Capture POC

A single static page (`index.html`). It opens the device camera (with a picker if there are several), finds faces in the live feed, saves one snapshot for each unique person and logs every time that person is seen. Everything runs in the browser. Nothing is uploaded.

**Live demo:** https://cheokjiade.github.io/video-face-detection/

## How it works

1. While the camera runs, the page copies the current frame onto a canvas every *N* seconds (0.5 s by default; 0 means as fast as the device can go).
2. It runs three models from [`@vladmandic/human`](https://github.com/vladmandic/human) 3.3.6 on that frame:
   - MediaPipe BlazeFace finds the faces.
   - MediaPipe FaceMesh lines each face up, using 468 points.
   - HSE FaceRes turns each face into a 1024-number descriptor.
3. It compares each descriptor with the faces already found, using Human's similarity score (0 to 1). A score of at least 0.5 (the default) means the same person, and the time is added to that person's list. Otherwise it saves a cropped snapshot as a new unique face.
   - **How 0.5 was chosen:** each Apollo 11 astronaut in the crew photo was compared with their own separate NASA portrait. Same person scored 0.61, 0.68 and 0.86; different people scored 0.18–0.40. Within one camera session, same-person scores should be higher still.
   - **Starting a new face:** a face only becomes a new card if the detection is at least 80% confident and the crop is sharp enough. Sharpness is the variance of the Laplacian on a 64×64 grey crop; the default minimum is 40. Detector confidence stays at 95–100% even on blurred faces, so confidence alone can't filter out blur. With face-api's tighter crops, a sharp photo scored about 2,700–3,900, Gaussian blur of radius 2 about 90, and radius 3 or more under 30. With Human, the sharp photo scores about 4,700–5,800; at blur radius 2–3 faces are still accepted, and at radius 6 they are rejected (sharpness about 29). The log shows each face's `confidence%/sSharpness` so you can tune the limits.
   - **One match per frame:** within a single frame, each known face can match only one detection. Blurred faces of different people look alike, and without this rule they would merge into one card.
   - **Best snapshot:** every card keeps its sharpest sighting. That sighting's image is shown, and its descriptor is used for later matching.
4. A **Performance** panel updates after every scan. It shows:
   - **OK/SLOW/CONTINUOUS**: whether scans take more than 60% of the scan interval, judged on the average of the last 20 scans (so it catches slowdown from heat). CONTINUOUS means the interval is 0.
   - **Worst scan** and total scan count.
   - **FPS**:
     - **Scanning**: scans actually completed per second.
     - **Device could do**: the scan rate the device could reach (1000 ÷ average scan time).
     - **Page**: screen refreshes per second. If this drops, the page is janky.
     - **Camera**: the frame rate the camera is delivering.
   - **Last scan split into steps**: copying the frame, running the models, and matching faces.
   - **Memory**:
     - **TensorFlow.js memory and tensor count**: if this keeps growing, memory is leaking.
     - **GPU memory**.
     - **JavaScript heap**: Chromium browsers only.
   - **System info**: backend, camera resolution, Human version, model load time and GPU warm-up time.
5. Right after loading, the page runs `human.warmup()` once. The GPU compiles its programs on a model's first run, which took about 15 s in testing with face-api, so this keeps the first camera scan from freezing.

**Backend:** Human picks one automatically. It prefers **WebGPU** where the browser supports it, and falls back to WebGL. To force one for comparison, add a URL parameter: `?backend=webgl` or `?backend=webgpu`.

To run it locally, run `npx serve` and open the localhost URL. The camera needs HTTPS or localhost, so opening the file directly may not work. The models load from the jsDelivr CDN.

## Hardware requirements

> **The device table below contains estimates.** To measure a real device, open the page on it and read the Performance panel.

**Measured with Human (current version):**

- **Desktop Edge**, on a 1280×1006 image of 3 faces:
  - **WebGPU**: averaged 135 ms per scan, worst 169 ms. GPU warm-up took 2.1 s.
  - **WebGL**: averaged 231 ms per scan, worst 513 ms. GPU warm-up took 2.9 s.
  - TensorFlow.js memory: 16.7 MB.
- **On this PC, Human is slower per scan than face-api was** (about 135 ms vs 55–100 ms). Its alignment tracks 468 points instead of 68, and its descriptor model works on larger images, so each face costs more.
- **The iPhone 15 Pro and MacBook Air M4 still need re-measuring** with Human.

**Measured with face-api (the previous version, for comparison):**

- **iPhone 15 Pro** (Chrome on iOS, so the WebKit engine; WebGL; 720×1280 camera, 1 face):
  - Averaged 109 ms per scan, worst 135 ms, over 72 scans (models about 114 ms, frame copy about 6 ms).
  - Memory: 44 MB. GPU warm-up: 0.8 s.
- **MacBook Air M4** (WebGL; 1280×720 camera, 1 face):
  - Averaged 81 ms per scan, worst 98 ms, over 38 scans (models about 58 ms, frame copy about 12 ms).
  - Memory: 44 MB. GPU warm-up: 2.3 s.
- **Desktop Edge** (WebGL; 1280×1006 image of 3 faces):
  - Averaged 55–100 ms per scan.
  - Memory: 44 MB. GPU warm-up: 1.3 s with the shader cache already filled; the first-ever run took about 15 s.

**What "smooth" means here:** each scan has to finish faster than the scan interval. With the default 0.5 s interval, a scan time under about 300 ms leaves room for the camera preview to stay smooth.

**Cost of one scan:** one face detection on the frame, plus alignment and a descriptor for *each* face. More faces in the frame means a longer scan.

| Device (released in the last 2 years) | Expected scan time (1–3 faces) | Verdict at 0.5 s interval |
|---|---|---|
| Laptop, Apple M-series or Intel Core Ultra / AMD Ryzen 7000+ with integrated GPU | ~50–150 ms | Smooth. Can scan every 0.2 s |
| Laptop with a discrete GPU | ~20–80 ms | Smooth |
| Flagship phone (iPhone 15/16, Pixel 8/9, Galaxy S24/S25) | ~100–300 ms | Smooth |
| Mid-range phone (e.g. Galaxy A5x, Pixel 8a) | ~250–600 ms | Usable. Set the interval to 1 s |
| Budget phone (under ~$200) | 600 ms–2 s+ | Choppy. Use a 1–2 s interval, or move detection to a server |

**Minimum requirements:**
- **Browser:** a current version of Chrome, Edge, Safari or Firefox with WebGPU or WebGL2. The Performance panel shows the backend; `cpu` will be very slow.
- **RAM:** about 200–400 MB for the browser tab. 4 GB of system RAM is enough. On phones, close other apps.
- **Download:** about 9 MB of model files on first load (FaceRes 6.7 MB, FaceMesh 1.5 MB, BlazeFace 0.5 MB), plus the Human library itself. After that the browser caches them.
- **Camera:** the page asks for 1280px wide. Every scan copies the full frame, so higher resolutions cost more memory and copy time and can stutter on phones.
- **Long sessions on phones:** the GPU running for a long time causes thermal throttling. Expect scans to get 1.5–2 times slower after a few minutes.

**When to move to a server:** if the target is low-end phones, or many faces per frame (crowds), run only face *detection* on the device (or none at all), and send the frames or crops to the planned API for the descriptor step.

## License report

> This is an engineering summary, not legal advice. Have counsel review it before any commercial or production use.

### Code and libraries

| Component | License | Notes |
|---|---|---|
| [`@vladmandic/human`](https://github.com/vladmandic/human) 3.3.6 | MIT | Maintained; last updated Dec 2025. It replaced [`@vladmandic/face-api`](https://github.com/vladmandic/face-api), which was archived in Feb 2025 |
| TensorFlow.js (bundled inside Human's `dist`) | Apache-2.0 | Keep the NOTICE/license if you redistribute it |
| This repository's own code | **No license chosen yet** | Without a LICENSE file, others may view the code but have no right to reuse it. Add MIT (or similar) if you want that. |

MIT and Apache-2.0 both allow commercial use. Their only real requirement is keeping the copyright and license notices.

### Model weights (the riskier part)

| Model | Source | License / status | Commercial risk |
|---|---|---|---|
| BlazeFace (finds faces) | Google MediaPipe | Apache-2.0 | Low |
| FaceMesh (aligns faces) | Google MediaPipe | Apache-2.0 | Low |
| FaceRes (descriptor) | [HSE_FaceRec_tf](https://github.com/av-savchenko/HSE_FaceRec_tf), Higher School of Economics | Repo is Apache-2.0. Identity training used **VGGFace2**: faces scraped from Google Images, and Oxford no longer distributes it. The author's combined age/gender models also used **UTKFace**, which is licensed for non-commercial research only | **Medium.** The code license is permissive, but the training data is scraped faces, and part of it is research-only. Whether dataset terms carry over to trained weights is legally unsettled |

**Compared with face-api:** finding and aligning faces is now clean (Apache-2.0, from Google). The landmark model with unknown training data, and the detector trained on research-use data, are both gone. The remaining question is the descriptor's training data.

**For commercial use**, either:
- get legal sign-off on FaceRes, or
- use a descriptor model trained on licensed or synthetic data, or a commercial SDK.

Human can also load MobileFaceNet or InsightFace instead of FaceRes, but InsightFace's pretrained weights are explicitly non-commercial.

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
