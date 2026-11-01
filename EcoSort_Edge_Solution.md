# EcoSort-Edge — Offline Smart Waste-Segregation Assistant
### Llama + ExecuTorch on Arm · One shared core · Raspberry Pi 5 *and* Android

> **Design thesis:** *Vision perceives, rules decide, Llama explains.*
> The LLM is never allowed to choose the bin. It explains, asks clarifying questions, and educates, and every answer passes a deterministic policy check. That is the main difference from a generic "VLM looks at trash" demo.

---

## 0. Numbers legend (read this first)

Every number in this document carries one of three tags, so nothing in the report can be accused of being invented.

| Tag | Meaning |
|---|---|
| **[REF]** | Published by a third party (ExecuTorch / Arm / Meta docs). Cite the source in your report. |
| **[TARGET]** | Design goal. You must verify it with your harness. |
| **[MEASURE]** | Blank cell. Fill it from your own benchmark run on your own device. |

Juries reward measured PPA, so the ablation tables below ship with a harness (§8) that fills every `[MEASURE]` cell.

**Policy update that affects the brief.** The brief cites the 2016 rules. The **Solid Waste Management Rules, 2026** were notified on 27 Jan 2026, took effect on **1 April 2026**, and supersede the 2016 rules. They mandate **four-stream segregation at source: wet, dry, sanitary, special-care**. Mapping your outputs to the 2026 streams, and not the old wet/dry/hazardous split, is a cheap differentiator. Re-verify the clause-level details against the official gazette before quoting any section number.

---

## 1. What the jury scores and how this design answers it

| Jury criterion | Design answer | Evidence you will show |
|---|---|---|
| PPA metrics (ms, tok/s, MB, W, °C) | Built-in `ppa_trace` in the core, identical on Pi and phone | Table + plots per stage |
| Edge decoupling | Perception pipeline and Cognition pipeline are separate modules with a typed contract (`Evidence` → `Advice`) | Architecture diagram, §2 |
| Arm hardware use | ExecuTorch **XNNPACK + KleidiAI** (int4-weight / int8-dynamic-activation), NEON/dotprod, big-core pinning, optional Hailo NPU on Pi | Backend ablation, §6 |
| Offline and private | Zero network calls in the core. CI test blocks sockets. | `strace`/airplane-mode demo |
| Indian context | Rule pack for SWM Rules 2026 four streams, E-Waste Rules 2022, Battery Waste Rules 2022, Plastic Waste Rules, Hindi output | Golden-set accuracy by stream |
| Required: Llama + ExecuTorch | Llama 3.2 1B (default) / 3B (pro mode) exported to `.pte`, run by the ExecuTorch LLM runner | Export recipe, §9 |
| Hailo separation (if used) | Hailo vision runs through HailoRT; Llama runs on the CPU through ExecuTorch. Documented as two pipelines. | §4.3 |

---

## 2. System architecture

```
                      ┌────────────────────────────── EcoCore (C++17, C ABI) ──────────────────────────────┐
 PLATFORM SHELLS      │                                                                                    │
 ┌──────────────┐     │  PERCEPTION PIPELINE (no LLM)                COGNITION PIPELINE (Llama)            │
 │ Android app  │     │  ┌────────┐  ┌──────────┐  ┌────────────┐    ┌───────────┐  ┌───────────────────┐  │
 │ Kotlin +     │◄───►│  │Capture │→ │Detector  │→ │Attribute   │ →  │ Evidence  │→ │ Policy Engine     │  │
 │ CameraX +JNI │     │  │+quality│  │(YOLO-n)  │  │head: material│   │ Fusion    │  │ (deterministic:   │  │
 └──────────────┘     │  │gate    │  │          │  │soiled/hazard│   │ temporal  │  │  stream + route)  │  │
 ┌──────────────┐     │  └────────┘  └──────────┘  └────────────┘    │ vote +    │  └─────────┬─────────┘  │
 │ Pi 5 service │◄───►│        backend: ExecuTorch-XNNPACK           │ calibrated│            │ PolicyDecision
 │ `ecod` + web │     │        or HailoRT (Pi + Hailo HAT)           │ confidence│  ┌─────────▼─────────┐  │
 │ UI / CLI     │     │                                              └─────┬─────┘  │ Retriever (local  │  │
 └──────────────┘     │                                                    │        │ BM25 + facts DB)  │  │
        ▲             │   instant "template card" (<300 ms) ◄──────────────┘        └─────────┬─────────┘  │
        │             │                                                              ┌─────────▼─────────┐  │
        │             │   streamed explanation  ◄────────────────────────────────────│ Llama 3.2 on      │  │
        │             │                                                              │ ExecuTorch        │  │
        │             │   ┌────────────────┐   ┌───────────┐   ┌─────────────────┐   │ (constrained JSON)│  │
        └─────────────│───│ Verifier gate  │◄──│ Advice    │◄──└─────────────────┘   └───────────────────┘  │
                      │   │ (policy-lock)  │   └───────────┘                                              │
                      │   └───────┬────────┘                                                              │
                      │           ▼                                                                       │
                      │   Impact Ledger · Speech (STT/TTS) · PPA tracer · Rule-pack loader                │
                      └────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Contracts between stages (this is the "well-defined pipeline")

```jsonc
// Evidence  (Perception → Cognition). Produced identically on Pi and phone.
{
  "frame_id": 1042,
  "items": [{
    "bbox": [x, y, w, h],
    "material":   {"label": "plastic_pet_bottle", "p": 0.91},
    "attributes": {"soiled": 0.12, "contains_liquid": 0.64, "hazard_flag": 0.01},
    "stability":  0.88,           // temporal-vote agreement over the last N frames
    "ood_score":  0.07            // open-set / out-of-distribution score
  }],
  "quality": {"blur": 0.1, "glare": 0.0, "too_dark": false}
}

// PolicyDecision (rules, no LLM)
{
  "stream": "dry",               // wet | dry | sanitary | special_care | reject_ask_user
  "route":  "kerbside_dry_bin",  // or "ewaste_takeback", "pharmacy_drop", "authorised_collector"
  "prep_steps": ["empty", "rinse", "crush", "cap_on"],
  "rule_ids": ["SWM2026.dry", "PWM.pet"],
  "confidence_band": "high",     // high | medium | low → drives UX
  "clarify": null                // or {"question_id": "Q_SOILED", "options": ["Clean","Greasy"]}
}

// Advice (Cognition output, schema-constrained)
{
  "headline": "...", "why": "...", "steps": ["..."], "impact_fact": "...",
  "follow_up": "...", "lang": "en"
}
```

### 2.2 Why this beats generic submissions

1. **Policy-locked generation.** The verifier rejects any Llama output whose stream/route/steps contradict the `PolicyDecision`, then falls back to a template. Policy consistency is **100 % by construction**, and you can report it.
2. **Attribute-aware perception.** Material is only half the answer. A greasy pizza box and a clean cardboard box go to different streams. The attribute head predicts *soiled / liquid / hazard*.
3. **Uncertainty becomes dialogue.** If vision is unsure, the system doesn't guess. The policy engine emits a `clarify` question, Llama phrases it naturally, and the user taps or says an answer. That turns a weak classifier into a multimodal interaction.
4. **Two-speed UX.** A deterministic **template card appears in under 300 ms [TARGET]**, and Llama's explanation streams in afterward. This hides LLM latency without faking it.
5. **Short, fixed prompt budget.** Prompts are capped near 250 tokens, so prefill is cheap. For reference, 1B SpinQuant on a OnePlus 12 prefills at ~260 tok/s **[REF]**, so a 250-token prompt costs about 1 s of prefill (derived).
6. **Offline Impact Ledger.** Per-item CO₂e/landfill-diversion estimates come from a lookup table (clearly labelled *estimates*) and accumulate into a household/community score.
7. **Signed, versioned rule packs.** Regulations change (they did in 2026). Rule packs are data files with a version and a signature. They can be updated offline by USB or QR without re-exporting any model.
8. **One benchmark harness for both platforms**, so the Pi and phone results are directly comparable.

---

## 3. Pipeline stages in detail

### Stage 0 · Capture and quality gate
- Downscale to the detector input (e.g., 320 or 416 px), with letterboxing in NEON-friendly NV12/RGB paths.
- Blur (variance of Laplacian), glare, and darkness checks. If the frame is bad, the UI says *"move closer / more light"* and **no inference is spent**.
- Motion gate: run the detector only when the scene has been stable for ~150 ms, so battery isn't wasted on a waving hand.

### Stage 1 · Detection
- **YOLO-nano class detector** (single or multi-item), exported through ExecuTorch (XNNPACK, int8 PT2E quantization) for Android and Pi-CPU. The same model is compiled to a **HEF** for the Hailo path.
- Output: boxes plus a coarse class. Crops go to Stage 2.

### Stage 2 · Attribute head (the part toy classifiers lack)
- A small shared backbone (MobileNetV3-small / EfficientNet-Lite0 class) with **multi-head output**: fine material (~25 classes), `soiled`, `contains_liquid`, `hazard_flag`.
- Calibrated with temperature scaling. Report **ECE** and show a reliability diagram.
- OOD score from max-logit/energy. High OOD means "I don't know this", not a confident wrong answer.

### Stage 3 · Evidence fusion
- Sliding window of N frames (default 5). Majority vote weighted by confidence, with stability = agreement ratio.
- Output the confidence band: `high / medium / low`.
  - **high**: show the answer immediately.
  - **medium**: show the answer plus a "not sure? tell me more" chip.
  - **low or OOD**: ask the clarifying question, or request a better angle.

### Stage 4 · Policy engine (deterministic, auditable)
- Maps `(material, attributes)` → `(stream, route, prep_steps)`.
- Rule pack format (YAML compiled to a compact binary or SQLite):

```yaml
- item: battery_dry_cell
  stream: special_care
  route: authorised_collector
  prep_steps: [tape_terminals_if_lithium, keep_dry, do_not_crush]
  rule_ids: [SWM2026.special_care, BWM2022]
  facts: [battery_fire_risk, metals_recovery]

- item: food_container_plastic
  when: {soiled: ">0.5"}
  stream: dry            # soiled recyclables: rinse first, otherwise contaminates the batch
  prep_steps: [scrape, rinse, dry]
  clarify_if_uncertain: Q_SOILED
```
- Streams follow SWM 2026: **wet** (kitchen/food), **dry** (plastic, paper, metal, glass), **sanitary** (diapers, napkins), **special-care** (bulbs, batteries, medicines, paints). E-waste maps to a take-back/authorised-collector route under the E-Waste (Management) Rules 2022. Have a domain reviewer confirm the exact mapping before submission.
- **Safety rule:** items flagged biomedical (e.g., used syringes) never get disposal instructions beyond "do not handle, contact the local body / authorised facility."

### Stage 5 · Retrieval (small, local, honest)
- Facts DB (~200–400 short, curated snippets: why PET is valuable, battery fire risk, composting steps, EPR basics).
- **BM25 + metadata filter by item id**. No vector DB is needed for a closed domain, which saves RAM and startup time. (Embedding retrieval is ablated in §6.)

### Stage 6 · Llama advisor (ExecuTorch)
- Prompt = system rules (~80 tokens) + `PolicyDecision` + ≤3 retrieved facts + user language/persona.
- **Constrained output**: JSON schema (via grammar/logit masking if your runner build supports it, otherwise strict prompt + parse + repair + verifier).
- Sampling: temperature ≈ 0.2, top-p 0.9, max_new_tokens ≈ 120. Short outputs matter for latency and battery.
- Conversational mode: the same path handles follow-ups ("Can I recycle pizza boxes?") with a rolling 2-turn memory and a hard context cap.

### Stage 7 · Verifier gate
Checks: schema valid · stream and route strings equal the `PolicyDecision` · every prep step ⊆ allowed steps · no numbers outside the facts DB · language is as requested. On failure: **one** retry with a stricter prompt, then the template fallback. Log the failure rate. It is a reportable metric.

### Stage 8 · Speech (optional, offline)
- STT: small Whisper-class or a lightweight streaming recognizer. TTS: platform TTS on Android; Piper-class on Pi.
- Hindi: pre-authored **template strings** are always available. Llama-generated Hindi is a switchable experiment (see ablation A7), because 1B-class Hindi quality is the weak link.

### Stage 9 · Impact ledger
- `kg_CO2e_avoided ≈ mass_estimate × factor(material)`. Mass comes from a per-class typical-mass table, not from vision. Label the output "estimate".
- Gamified streaks and a community leaderboard (local only) support the *promote sustainable practices* requirement.

---

## 4. Platform shells over the shared core

### 4.1 What is shared (≈ 85–90 % of code)
Perception orchestration, fusion, policy engine, retriever, prompt builder, verifier, impact ledger, rule packs, PPA tracer, model files (`.pte`), and the C ABI.

```c
// ecocore.h  (stable C ABI → trivially callable from JNI, Python, Go, anything)
typedef struct ec_ctx ec_ctx;
ec_ctx* ec_create(const ec_config* cfg);                 // model paths, threads, backend, rule pack
int     ec_submit_frame(ec_ctx*, const ec_frame*);       // returns when Evidence is ready
int     ec_get_decision(ec_ctx*, ec_decision* out);      // instant template card
int     ec_advise_async(ec_ctx*, const ec_query*, ec_token_cb, void* user);  // streamed Llama
int     ec_ppa_dump(ec_ctx*, const char* json_path);
void    ec_destroy(ec_ctx*);
```

### 4.2 Android shell
- Kotlin UI, CameraX `ImageAnalysis` (YUV) → JNI → `ec_submit_frame`.
- Foreground `ThermalStatus` listener: at `THERMAL_STATUS_MODERATE` drop to 3 threads and a lower detector rate. At `SEVERE`, pause the LLM and use template-only mode.
- Pin inference threads to the big cores. Record battery drain per 100 interactions.

### 4.3 Raspberry Pi 5 shell
- `ecod` C++ daemon: libcamera/V4L2/OpenCV capture → core. Local web UI (kiosk Chromium) over WebSocket on `127.0.0.1`, plus a CLI. A physical push-button and a status LED stay responsive even when the LLM is busy.
- **Variant A, CPU-only:** detector + attribute head + Llama all on ExecuTorch-XNNPACK. Cortex-A76 has dot-product but **no i8mm**, so expect lower LLM throughput than a recent flagship phone.
- **Variant B, Pi 5 + Hailo AI HAT+:** two *separate* pipelines. Document both explicitly, as the brief requires.

```
Vision pipeline :  camera → HailoRT (HEF on Hailo NPU) → Evidence
LLM pipeline    :  PolicyDecision → ExecuTorch (XNNPACK/KleidiAI on Arm CPU) → Advice
                   (no tensors cross between them, only the typed Evidence JSON)
```
- Benefit to measure: the NPU frees CPU cores for Llama. Ablate *sequential vs concurrent* scheduling (A9).

### 4.4 Fallback if the NDK build of the shared core blocks you
Use the ExecuTorch Android AAR (`LlmModule`) *inside the Android shell only*, keeping policy/verifier/retriever in the shared core. Say so openly in the report as a tiered-risk decision.

---

## 5. Taxonomy (what the model recognises)

~25 fine classes → 4 SWM-2026 streams + routes. Suggested set:

| Stream | Fine classes (examples) |
|---|---|
| **Dry** | PET bottle, HDPE container, plastic film/carry-bag, multilayer sachet (low-value; flag as hard-to-recycle/EPR), PP cup/tub, thermocol (EPS), paper, cardboard, tetra-pack, glass bottle, aluminium can, steel can, textile, rubber/footwear |
| **Wet** | cooked food, fruit/veg peels, garden waste, tea/coffee grounds |
| **Sanitary** | diaper, sanitary napkin, mask, cotton/bandage (household) |
| **Special-care** | battery, CFL/LED bulb, medicine strip/expired medicine, paint can, aerosol, thermometer (mercury) |
| **E-waste route** | phone, charger/cables, earphones, power bank (also flagged fire risk) |
| **Escalate / don't handle** | syringe/sharps, unknown chemical container |

**Data sources to combine (verify each licence before use):** TACO, TrashNet, public e-waste and garbage-classification datasets from Kaggle/Roboflow, **plus your own 1.5–3 k Indian-context photos** (street, kitchen, bin, kerbside lighting, crushed and soiled items). The in-house Indian set is your moat; report its size and collection protocol.

---

## 6. Tech-stack ablations (each choice vs. the alternatives)

> Columns marked **[MEASURE]** are produced by the harness in §8. The "Expected" column is a hypothesis to confirm, not a result.

### A1 · Reasoning model
| Option | Fits "Llama required"? | Expected on-device cost | Quality for grounded advice | Verdict |
|---|---|---|---|---|
| Llama 3.2 **1B** Instruct (8da4w) | ✅ | Lowest RAM/latency; **[REF]** ~50 tok/s decode, ~1.9 GB RSS on OnePlus 12 (SpinQuant) | Adequate *because* output is grounded and constrained | **Default** |
| Llama 3.2 **3B** Instruct (8da4w) | ✅ | ~2–3× slower decode, larger RSS | Better phrasing, Hindi, follow-ups | **"Pro mode"** on flagship phones / Pi 16 GB |
| Llama 3.1 8B | ✅ | Too heavy for Pi 5 CPU interaction | Overkill | Rejected (latency, RAM) |
| Llama 3.2 11B Vision (end-to-end VLM) | ✅ | Far beyond edge budget for this use | Redundant with the cascade | Rejected |
| Non-Llama small LMs | ❌ violates rules | n/a | n/a | Excluded by brief |

**Metric to run:** TTFT, decode tok/s, peak RSS, JSON-valid rate, faithfulness score, on the 300-prompt golden set.

### A2 · Quantisation
| Option | Notes | Expected | Verdict |
|---|---|---|---|
| BF16/FP32 baseline | Reference **[REF]**: 1B bf16 ≈ 19 tok/s decode, ~3.2 GB RSS (OnePlus 12) | Slowest, biggest | Baseline only |
| **8da4w** (int4 group-wise weights, int8 dynamic activations); SpinQuant or QAT+LoRA checkpoints | Native fit for KleidiAI kernels | **[REF]** ~50 tok/s, ~1.9 GB RSS for 1B SpinQuant | **Default** |
| int8 weight-only | Larger, slower than 8da4w | **[MEASURE]** | Rejected unless accuracy gap is shown |
| Post-hoc 4-bit weight-only (no activation quant) | Less kernel support in XNNPACK | **[MEASURE]** | Rejected |
| 2–3-bit | Quality cliff on small models | **[MEASURE]** | Rejected |

### A3 · Runtime (required: ExecuTorch)
Compare ExecuTorch against llama.cpp (GGUF Q4_K_M), ONNX Runtime, and TFLite **as a sanity benchmark only**, and report it honestly. Winner for the project is ExecuTorch because the brief requires it, and because **one runtime for LLM + vision** simplifies the shared core (single `.pte` loader, single profiler, uniform memory planning). Show the PPA numbers anyway. Juries like seeing you tested the competition.

### A4 · Backend and delegates
| Backend | Portability | Arm relevance | Verdict |
|---|---|---|---|
| ExecuTorch **portable** (reference ops) | Everywhere | None (slow) | Baseline |
| **XNNPACK (+KleidiAI micro-kernels)** | Any arm64 phone/Pi | NEON/dotprod/i8mm (and SME2 on newest cores) paths | **Default for LLM + vision** |
| Vendor NPU delegates (QNN, MediaTek, etc.) | Per-SoC | Not "Arm" per se; needs per-device compilation | Stretch goal, not the headline |
| Vulkan GPU | Device-specific | Mixed benefit on mobile for LLM decode | Not used |
| **HailoRT** (Pi HAT) | Pi + Hailo only | Offloads vision | **Used for vision on Pi Variant B** |

**Ablation to show:** portable → XNNPACK → XNNPACK+KleidiAI (and thread count 1/2/3/4) on tok/s and energy per answer.

### A5 · Architecture: cascade vs end-to-end VLM
| Option | Latency/energy | Context & policy accuracy | Verdict |
|---|---|---|---|
| Raw frame → on-device VLM | Worst; the brief itself warns of multi-second-per-token behaviour and thermal throttling | Free-form, unaudited | Rejected |
| Classifier only ("Plastic 82 %") | Fast | No context, no explanation, no Llama use | Rejected (fails requirements) |
| **Detector + attribute head → Policy → Llama (cascade)** | Vision in tens of ms; Llama only on demand | Auditable, grounded, uncertainty-aware | **Chosen** |

### A6 · Detector and attribute backbone
| Candidate | Why considered | Expected | Verdict |
|---|---|---|---|
| YOLO-nano class (e.g., v8n/11n) | Mature export path, HEF-friendly | Fast; good small-object recall | **Detector** |
| SSD-MobileNet / EfficientDet-Lite | Classic mobile detectors | Older, often lower accuracy per ms | Compare |
| Classifier-only MobileNetV3 | Smallest | No localisation, one item per frame | Used as attribute head backbone |
| ViT-tiny / DeiT-tiny | Strong accuracy | Heavier on CPU, no NEON-friendly speedups | Compare, probably reject |

**Check:** mAP50, macro-F1, ms/frame (p50/p95) at fp32 vs PT2E-int8, on Pi CPU, phone CPU, and Hailo. Report int8 accuracy drop.

### A7 · Languages
| Option | Result expected | Verdict |
|---|---|---|
| English via Llama | Best | Default |
| Hindi via Llama 3.2 1B | Weak fluency risk | Switchable experiment, quality scored by a native speaker |
| Hindi via **pre-authored templates** | Perfect on core advice, no follow-ups | **Always-on fallback** |
| Hindi via 3B | Better | Pro mode |

### A8 · Knowledge grounding
| Option | RAM/latency | Update path | Hallucination | Verdict |
|---|---|---|---|---|
| Fine-tune facts into Llama | Zero runtime cost | Needs re-export for every rule change | Higher | Rejected |
| **Rules engine + small BM25 facts DB** | Tiny | Replace data file | Lowest | **Chosen** |
| Embedding retrieval (small encoder) | Extra model, extra RAM | Reindex | Similar to BM25 in a closed domain | Ablate; adopt only if recall@3 improves materially |
| Long context dump | Prefill-heavy | n/a | Medium | Rejected |

### A9 · Pi scheduling (CPU-only vs Hailo)
Sequential vs concurrent vision/LLM, thread counts, and CPU governor. Metric: **end-to-end latency**, CPU temp, and `throttled` flags over a 30-minute soak.

### A10 · Shared-core language
| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **C++17 + C ABI** | ExecuTorch runtime is C++; JNI is simple; runs unchanged on Pi | Manual memory discipline | **Chosen** |
| Rust core | Safety | Extra FFI layer to ExecuTorch C++ | Rejected (integration cost) |
| Kotlin Multiplatform | Nice for UI | Native inference still needs C++ | Rejected |
| Python on both | Fast to write | No good Android story; slower startup; heavier RAM | Used only for training/export/tools |

### A11 · Generation control
Free text vs JSON-schema constrained vs template-only. Metrics: schema-valid %, verifier-reject %, tokens generated, latency. Expect constrained JSON plus verifier to dominate on reliability at a small token cost.

### A12 · Uncertainty strategy
Always-answer vs threshold-abstain vs **abstain + clarify question**. Metric: *stream-accuracy at coverage* curve, plus user-task success in a small usability test.

---

## 7. Latency, memory, and energy budgets

| Stage | Pi 5 CPU-only | Pi 5 + Hailo | Phone (recent Arm) | Tag |
|---|---|---|---|---|
| Capture + gate | ≤ 20 ms | ≤ 20 ms | ≤ 15 ms | TARGET |
| Detector + attribute head | ≤ 120 ms | ≤ 40 ms | ≤ 50 ms | TARGET |
| Fusion + policy + template card | ≤ 10 ms | ≤ 10 ms | ≤ 10 ms | TARGET |
| **Time to first visible answer** | **≤ 300 ms** | **≤ 200 ms** | **≤ 150 ms** | TARGET |
| Llama TTFT (≈250-token prompt) | **[MEASURE]** | **[MEASURE]** | ~1 s on a OnePlus 12-class device (derived from **[REF]** prefill) | |
| Llama decode | **[MEASURE]** | **[MEASURE]** | **[REF]** ~50 tok/s (1B SpinQuant, OnePlus 12) | |
| Full explanation (~100 tokens) | **[MEASURE]** | **[MEASURE]** | ~3 s (derived) | |
| Peak RSS, whole app | ≤ 2.5 GB | ≤ 2.5 GB | ≤ 2.5 GB | TARGET |

Set the Pi target after you measure. The A76 core has no i8mm, so don't copy phone numbers onto the Pi.

---

## 8. Benchmark and evaluation harness

**Build this first, before the app.** It is the single biggest score lever.

### 8.1 Performance protocol
- Fixed device state: charged, 25 °C ambient, screen brightness fixed, airplane mode, same thermal start temperature.
- 5 warm-up runs, then ≥ 30 measured runs. Report **median and p95**, not the mean.
- Fixed prompt set (the golden set), fixed seeds, `max_new_tokens` fixed.
- Metrics: TTFT, prefill tok/s, decode tok/s, e2e latency, peak RSS, model load time, CPU/NPU utilisation, energy/answer, temperature curve over a 30-minute soak.
- **Pi measurements:** USB-C power meter (or inline shunt) for watts, `vcgencmd measure_temp` and `vcgencmd get_throttled` for thermals, `perf`/`top` for CPU.
- **Android measurements:** `adb shell dumpsys batterystats`, Perfetto traces, `PowerManager.getCurrentThermalStatus`, `dumpsys meminfo`.

### 8.2 Accuracy protocol
| Layer | Metric | Target |
|---|---|---|
| Detector | mAP50 | set after baseline |
| Attribute head | macro-F1 per class, per-stream accuracy, **ECE** | stream-accuracy [TARGET] ≥ 90 % on held-out Indian set |
| End-to-end | stream correctness @ coverage, abstain rate | report the curve |
| Llama advice | JSON-valid %, verifier-reject %, faithfulness (checklist vs. retrieved facts), Hindi quality (native reviewer) | policy-consistency 100 % |
| Robustness | accuracy under low light, blur, partial occlusion, clutter | report drop |

Split by **location/collection session**, not randomly, to avoid leakage between near-duplicate photos.

### 8.3 Output artefacts
`bench/results/*.json` → auto-generated plots and tables → pasted into the technical report. Every ablation table in §6 gets a **[MEASURE]** column filled this way, with commit hash and device model recorded.

---

## 9. Build and export recipes

> Command names and flags change between ExecuTorch releases. Check the docs for the version you pin, and record that version in the report.

**LLM export (conceptual steps):**
1. Pick the Llama 3.2 1B/3B Instruct checkpoint (SpinQuant or QAT+LoRA quantised weights where available, or quantise to 8da4w yourself).
2. Export with the ExecuTorch LLM export tool, with the **XNNPACK partitioner**, a **fixed context length** (e.g., 2048 is plenty for this app, and it bounds the fp32 KV-cache allocated at load).
3. Output `llama3_2_1b_8da4w.pte` plus the tokenizer.
4. Build the runtime for `arm64-v8a` (Android NDK) and `aarch64-linux` (Pi), with **KleidiAI enabled** in the XNNPACK build option. Confirm at run time that the KleidiAI path is active, because a silent fallback is the most common reason for slow numbers.

**Vision export:** train → PT2E int8 quantisation with a representative calibration set → ExecuTorch XNNPACK `.pte`. For Hailo: ONNX → Hailo Dataflow Compiler → `.hef`.

**Repo layout**
```
ecosort-edge/
├─ core/            # C++17: perception/, fusion/, policy/, retrieval/, llm/, verifier/, ppa/
├─ rulepacks/       # swm2026.yaml, ewaste2022.yaml, battery2022.yaml, hi/ templates
├─ models/          # *.pte, *.hef, tokenizer  (git-lfs or release assets)
├─ android/         # Kotlin app + CMake/JNI
├─ pi/              # ecod daemon, web UI, systemd unit, Hailo adapter
├─ training/        # datasets, train, calibrate, export scripts (Python)
├─ bench/           # harness, golden sets, results, plotting
├─ tests/           # no-network test, verifier tests, rulepack lint
└─ docs/            # architecture, ablations, technical report
```

---

## 10. Risks and mitigations

| Risk | Mitigation |
|---|---|
| KleidiAI not actually used | Startup self-check plus a benchmark that flags it; compare against baseline in the report |
| Pi thermal throttling | Heatsink/fan, cap threads, duty-cycle the detector, watch `get_throttled` |
| NDK build of the shared core stalls | Fallback in §4.4 |
| Llama says something wrong | Verifier gate, template fallback, no LLM-chosen bins |
| Hindi quality | Template-first Hindi; Llama Hindi optional |
| Dataset domain gap | Own Indian dataset, session-based split, report robustness drop |
| Regulation drift | Versioned rule packs; a documented update path |
| Safety-sensitive items | "Do not handle" route, no instructions beyond escalation |
| Over-claiming | Use the §0 tags. Only cite numbers you measured. |

---

## 11. Phased plan

| Phase | Deliverable | Exit test |
|---|---|---|
| P0 | Harness and golden set skeleton, rule packs v0 | Harness runs a dummy model on both devices |
| P1 | Vision (detector + attribute head) trained and exported; ECE measured | Per-stream accuracy report |
| P2 | Policy engine + template card, **end-to-end without LLM** | Demo works offline in < 300 ms target |
| P3 | Llama on ExecuTorch + verifier + constrained JSON | Policy-consistency 100 %, JSON-valid ≥ 98 % |
| P4 | Android shell and Pi shell (CPU-only), same core | Same golden set gives the same decisions on both |
| P5 | Hailo variant, thermal soak, ablations A1–A12 filled | All `[MEASURE]` cells replaced |
| P6 | Video, report, polish | Dry run with a stranger, in airplane mode |

---

## 12. Demo video script (≈ 4 min)

1. **0:00** Airplane mode on. Show the network indicator off on both devices.
2. **0:20** Phone: scan a plastic bottle → card in < 0.3 s → Llama explanation streams → impact counter ticks up.
3. **1:00** Phone: greasy pizza box → system asks *"Is it greasy?"* by voice → user answers → different stream, with a reason.
4. **1:45** Pi 5: same two items, same answers, with the live PPA overlay (ms, tok/s, MB, °C).
5. **2:30** Battery and e-waste items → special-care/take-back route, citing the **SWM Rules 2026** four streams.
6. **3:00** Pi + Hailo: show the two separate pipelines in the dashboard; CPU freed for Llama.
7. **3:30** Ablation slide: portable → XNNPACK → KleidiAI, and cascade vs VLM.
8. **3:50** Verifier catching a deliberately corrupted Llama output. Call it out on screen.

---

## 13. Technical report outline

1. Problem and Indian regulatory context (SWM Rules 2026 four streams; E-Waste and Battery rules)
2. Architecture: perception/cognition decoupling and the typed contracts
3. Shared-core design (C ABI) and the platform shells
4. Models, data, training, and calibration
5. Policy engine and verifier ("policy-locked generation")
6. **Ablations A1–A12** with measured numbers
7. PPA results: Pi CPU-only, Pi + Hailo, Android
8. Accuracy, robustness, and failure analysis
9. Limitations (honest) and future work
10. Reproduction guide: versions, commands, hashes

---

## 14. Final checklist before submission

- [ ] Llama model runs through the **ExecuTorch** LLM runner on both devices (screenshots + logs)
- [ ] KleidiAI path confirmed active
- [ ] No network calls in the core (automated test)
- [ ] Every `[MEASURE]` cell filled, with device, commit, and ExecuTorch version
- [ ] Hailo and Llama pipelines documented as separate (if Hailo is used)
- [ ] Rule pack cites the SWM Rules 2026 and has been reviewed for accuracy
- [ ] Safety-sensitive items handled by escalation only
- [ ] Video shows airplane mode, both devices, and the PPA overlay
- [ ] Licence audit of datasets, models, and the Llama community licence
