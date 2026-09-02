# HQ

Camera-powered web app that snaps a photo and tells you, in plain language, everything it can find in it — mostly entirely in the browser, with an optional online double-check against Wikipedia.

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
6. **Optionally verifies the top guess against the real world** using Wikipedia:
   - After analysis, HQ can look up its best guess (e.g. "golden retriever", "espresso machine") on Wikipedia and show you the reference photo, a short description, and a link to read more
   - It then runs that reference photo through the same on-device MobileNet model used for detection and computes a **visual similarity score** between your capture and the reference image — a real, computed number, not a guess
   - You can also pick any other detected guess from the dropdown, or type your own term, and re-run the comparison manually
   - This step is **opt-in and toggleable** ("Auto-verify top guess online") — turn it off to keep the whole session fully offline as before
   - Only a short text label is ever sent anywhere (to Wikipedia's public API); your photo itself is never uploaded — the similarity comparison happens on-device, in your browser
7. **Explains its reasoning, honestly**:
   - Every localized (COCO-SSD) category shows a "Typically identified by…" note — a short, curated description of that category's usual visual traits (shape, texture, parts). This is general domain knowledge added on top of the model, **not** something extracted from the network's internals — neural nets like these don't expose human-readable rules, so nothing claims otherwise
   - Every broader (MobileNet) guess has a "🔥 Why?" button that computes a **real occlusion heatmap**: it hides one region of your photo at a time, re-runs the classifier, and measures how much each region's removal drops that guess's score. Regions that mattered most are highlighted in red. This is measured evidence from your actual photo, not a simulated or generic explanation
   - COCO-SSD's bounding boxes already show *where* it detected something, so the heatmap is offered only for the boxless broad guesses, where there'd otherwise be no spatial evidence at all

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
- All object detection happens client-side (TensorFlow.js + COCO-SSD, `mobilenet_v2` base for higher accuracy, with automatic fallback to `lite_mobilenet_v2` on slow connections)
- No backend or API keys required — the web-verification step calls Wikipedia's free, keyless public API (`en.wikipedia.org/api/rest_v1` and `en.wikipedia.org/w/api.php`) directly from the browser
- `vercel.json` ships a Content-Security-Policy that explicitly allow-lists the domains the app actually talks to: `cdn.jsdelivr.net` and `storage.googleapis.com` for the TF.js library/models, and `en.wikipedia.org` + `upload.wikimedia.org` for the optional verification step — plus a `Permissions-Policy` that allows the camera but blocks microphone/geolocation, which this app never needs
- If you fork this for a different reference source (Wikidata, iNaturalist, a product database, etc.), remember to update the CSP `connect-src`/`img-src` allow-list in `vercel.json` to match, or the browser will silently block the new requests
- The similarity score is a cosine similarity between MobileNet embeddings of your photo and the fetched reference photo — it's a genuine on-device computation, but it's still a whole-image embedding, not object-level matching, so busy or multi-object scenes will score lower even when the label is correct
- The "Typically identified by" text is a hand-written reference for COCO-SSD's fixed 80 classes — it describes what that category usually looks like in general, it is **not** a readout of what the neural network actually computed for your specific photo
- The occlusion heatmap for broad guesses runs 25 extra on-device classifications (a 5×5 grid) per "Why?" click, so it takes a couple of seconds — it's real measured sensitivity, but at 5×5 resolution it's coarse, and very small or thin objects may not align cleanly with grid cells
