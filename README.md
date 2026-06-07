# ListingLens 🏠🔍
**An eval harness for AI listing-photo tagging.**

ListingLens is a small tool that extracts structured listing data from interior room photos using a vision LLM — and, more importantly, an **eval harness** that measures whether that AI output can actually be trusted. It tags each photo with room type, style, key objects, condition, a short description, and the model's own confidence, then scores every output against a hand-built golden set of 30 images.

> Built as a weekend project to demonstrate eval-first thinking for AI products. The interesting work isn't the tagging — it's measuring how often, and *where*, the model is wrong.

---

## Why this exists

AI room-tagging only powers good listing search if it's accurate. "It looked fine when I tried it" isn't a launch criterion — so before trusting the model, I measured it: where it's right, where it fails, and whether its confidence means anything. The goal was to treat a probabilistic feature like the probabilistic system it is.

---

## What I built

1. **A tagging tool** — sends each interior photo to a vision LLM and gets back strict JSON: `room_type`, `style_tags`, `key_objects`, `condition`, `description`, `is_a_room_photo`, and a self-reported `confidence` (0–100).
2. **An eval harness** — runs the tool across all 30 images, scores the output against a hand-built golden set, and reports accuracy, object precision/recall, confidence calibration, latency, and a list of failures.

**Stack:** Python (Google Colab) · Google Gemini vision API (`gemini-2.5-flash` / `flash-lite`) · pandas · golden set authored in Google Sheets.

---

## The golden set (the part that matters)

30 hand-labelled images, chosen deliberately:

- **20 "normal" rooms** — living rooms, bedrooms, kitchens, bathrooms, dining rooms, home offices, balconies.
- **10 edge cases designed to break the model** — an empty room, a dark photo, a cluttered multi-purpose room, an exterior (wrong subject), a floorplan (wrong input type), a blurry shot, a room mid-renovation, an extreme style, a mirror reflection, and a watermarked image.

For each image I wrote the expected answer *and*, for the edge cases, the **expected behaviour** — e.g. an empty room must not be given hallucinated furniture; a floorplan must be refused, not tagged; a blurry photo should come back with low confidence rather than a confident guess. The ground truth is human-authored on purpose: an eval is only as trustworthy as its reference answers.

---

## Results (30/30 images scored)

| Metric | Result |
|---|---|
| Is-it-a-room accuracy | **97%** (29/30) |
| Room-type accuracy (nameable types) | **90%** |
| Edge-case handling vs expected behaviour | **100%** (10/10) |
| Condition accuracy | 70% |
| Object precision / recall / F1 | 45% / 67% / 54% |
| Average / max latency | 3.4s / 11.1s |
| Confidently-wrong outputs (conf ≥ 85 yet incorrect) | 2 |

**Confidence calibration**

| Confidence bucket | Outputs | Accuracy |
|---|---|---|
| 0–59 | 2 | 100% |
| 60–84 | 1 | 100% |
| 85–100 | 27 | 93% |

---

## Key findings

**1. The model's failure mode is inverted from what you'd expect — and that's the headline.**
On genuinely hard images it correctly signalled uncertainty: the blurry photo came back at confidence **10**, the mirror room at **30**, the dark room at **75**. But both of its outright errors happened on *ordinary-looking* photos where it was fully confident — it rejected a balcony as "not a room" at **confidence 100**, and called a living room a dining room at 95. So the dangerous errors don't live in the weird images (the model knows those are hard); they hide in confident, normal-looking outputs — exactly where a human reviewer wouldn't think to look. A confidence threshold alone would catch the easy-to-spot cases and miss these two.

**2. Edge-case behaviour was solid.** The model didn't invent furniture in the empty room, correctly refused both the exterior and the floorplan as non-rooms, dropped its confidence on the blurry and mirror images, and even identified "under renovation" as a condition rather than calling a half-built kitchen "furnished."

**3. Object tagging is high-recall, lower-precision.** It found about two-thirds of the objects I listed (recall 67%) but over-lists — adding real-but-extra detail like "range hood" or "dishwasher." That drags string-matched precision down to 45% even though most extras are correct, which is itself a lesson: object-level eval needs human review or embedding-based matching, not exact string comparison.

**4. Condition is the weakest field (70%).** The model leans heavily on "furnished" and is inconsistent on finer labels like "staged" and "semi-furnished" — a clear target for prompt refinement.

---

## Confidently-wrong examples

*(Add screenshots — these are the most important images in the repo.)*

- **img20** — labelled *balcony*; model returned *"not a room (exterior)"* at **confidence 100**. `[screenshot]`
- **img02** — labelled *living room*; model returned *dining room* at confidence 95 (a genuinely multi-function space). `[screenshot]`

---

## Human-in-the-loop design

Based on the findings, outputs are routed to human review when **any** of these is true:
- confidence is below 75, **or**
- `is_a_room_photo` is false, **or**
- condition is `under_renovation`.

But finding #1 means a confidence threshold isn't enough on its own — confident errors won't trip it. So the design also spot-checks a random sample of high-confidence outputs, since that's where the silent failures hide.

---

## Limitations & what I'd do next

- The golden set is 30 images — enough to find real patterns, not to publish numbers. I'd grow it and balance it across room types.
- Object scoring is string-based; I'd move to embedding-based matching so "sofa" and "sectional sofa" count as a hit automatically.
- One model, one prompt. I'd A/B two prompts and run the same harness against a second model (e.g. an Anthropic or OpenAI vision model) to compare accuracy, cost, and calibration.
- Add the downstream step the real product needs (room render or listing match) and eval that too.

---

## How to run

1. Add your Gemini API key as a Colab secret named `GEMINI_API_KEY`.
2. Put your images and `golden_set.csv` in a Google Drive folder.
3. Run the notebook top to bottom: it tags all images → saves `ai_results.csv` → scores against the golden set → prints the scorecard.

**Links:** `[live demo]` · `[notebook / repo]` · `[2-min Loom walkthrough]`

---

*Built by [Apoorva Shakya] · [LinkedIn] · [email]*
