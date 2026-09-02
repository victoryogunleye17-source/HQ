# HQ

Camera-powered web app that snaps a photo and tells you, in plain language, everything it can find in it — entirely in the browser.

## What it does

1. **Asks for camera permission** with a clear overlay and helpful error messages if it's denied or unavailable.
2. **Shows a live camera preview**, with a switch-camera button if your device has more than one camera (front/back).
3. **Snaps a photo** on demand and freezes the frame.
4. **Immediately analyzes the captured image** using TensorFlow.js + COCO-SSD, entirely on-device (nothing is uploaded).
5. **Runs two on-device models and tells you everything it detected**, in plain language:
   - **COCO-SSD** (80 categories) — localizes objects with bounding boxes and per-instance confidence
   - **MobileNet/ImageNet** (1000 categories) — a broader whole-image classifier that catches things outside COCO-SSD's narrow 80-category vocabulary
   - A summary sentence ("14 objects detected across 5 categories…")
   - Per-category counts and confidence ranges, color-coded and bar-charted, split into "Localized objects" (has a position) and "Broader guesses" (whole-image, no position)
   - Bounding boxes with labels drawn directly on the captured photo
   - A "show every individual detection" list with a confidence score per object
   - **Honesty caveats**: if the two models disagree on the main subject, or only a single low-context region matched, the app flags this explicitly instead of presenting a guess as fact
   - Low-confidence guesses are separated out and clearly flagged
   - A "Detection detail" selector to trade off sensitivity vs. precision — set to **Maximum** or **High** to catch as many of the up-to-100 object instances per shot as possible
   - A "Save Photo" button to download the annotated image

**Known limits (read this before reporting "wrong" results):** Both models are small, offline, and trained on fixed category lists — COCO-SSD's 80 classes and MobileNet's 1000 ImageNet classes. If the real object isn't a reasonable match for *any* category in either list (e.g. a niche accessory, packaging, or close-up macro shot with little context), the models will still return a confident-sounding percentage for their closest guess — they cannot say "I don't recognize this." That's a hard limitation of running vision fully offline with no backend/API, not a bug. The disagreement/caveat flagging above is designed to surface exactly this situation rather than hide it. "Up to 100 objects in a shot" means up to 100 individual object *instances* COCO-SSD can flag at once (e.g. 6 people + 4 chairs + 3 cups...), not 100 different categories.

## Deploy to Vercel (recommended)

### Option 1 – Vercel CLI (fastest)

```bash
# Install Vercel CLI if you don't have it
npm i -g vercel

# From this folder
vercel
```

Follow the prompts. It will give you a live URL.

### Option 2 – GitHub + Vercel Dashboard

1. Push this folder to a GitHub repository
2. Go to [vercel.com](https://vercel.com) → **Add New Project**
3. Import the repository
4. Click **Deploy** (no build settings needed)

### Option 3 – Drag & Drop

1. Go to [vercel.com/new](https://vercel.com/new)
2. Drag the entire folder onto the page

## Local development

```bash
# Simple static server
npx serve .
# or
python3 -m http.server 3000
```

Open `http://localhost:3000` (camera requires HTTPS or localhost).

## Notes

- Camera access only works on `localhost` or HTTPS
- All processing happens client-side (TensorFlow.js + COCO-SSD, `mobilenet_v2` base for higher accuracy, with automatic fallback to `lite_mobilenet_v2` on slow connections)
- No backend or API keys required
