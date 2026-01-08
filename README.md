# Vision Quality Router Suite (Open WebUI)

A small, practical suite for Open WebUI that adds **robust image handling** to any text LLM via:
- **Vision side-call** (Ollama VLM)
- **Quality gate + retry**
- **Optional OCR-only pass**
- **Verifier pass** (grounding / hallucination-risk check)
- **Clear “ask questions instead of hallucinating” behavior**
- Optional follow-up pipes for **Vision** and **Graph/Diagram structure extraction**

Designed for users who prefer **accuracy and clarification** over fast but unreliable vision guesses.

## Provenance
This project is **inspired by** the Open WebUI community “Dynamic Vision Router” concept (image detection + routing).
This suite is an independent implementation that extends the idea with quality gating, OCR-only fallback, verifier pass,
EU language presets, and follow-up pipes.

## Credits
- **cyberjaeger** (author)
- ChatGPT (OpenAI) — implementation assistance
- Inspired by the Dynamic Vision Router concept in the Open WebUI community

---

## Components

### 1) Filter — Dynamic Vision Router Max (Vision Quality Router)
**What it does**
- Detects images in the last user message
- Calls an Ollama vision model (side-call)
- Enforces structured output: `SUMMARY / DETAILS / OCR / UNCERTAINTY`
- Scores quality; retries once if weak (optional)
- If OCR seems missing and the user likely asked “what does it say”, runs an OCR-only pass (optional)
- Runs a verifier pass to estimate grounding / hallucination risk (optional / conditional)
- Injects the final vision bundle into the conversation as a **system** message
- If quality gate trips, instructs the main model to ask **1–2 targeted questions** instead of hallucinating

**Output injection**
- Adds `VISION ANALYSIS` block + `VERIFIER RESULT` block into context.
- Does **not** change the main model. The text LLM remains the answering model.

### 2) Pipe — Vision Follow-up Pipe
**What it does**
- When the main model indicates missing/unclear details, this pipe can run a second targeted vision query focusing only on missing elements.
- Appends a `VISION_FOLLOWUP_RESULT` message into the chat.

### 3) Pipe — Graph Follow-up Pipe
**What it does**
- When the image is a diagram/flowchart/graph, this pipe requests **structure extraction**
  (nodes, edges, labels, legend) with minimal interpretation.
- Appends a `GRAPH_FOLLOWUP_RESULT` message into the chat.

---

## Architecture (high-level)

Goal: add reliable vision to any text LLM without switching the main answering model.

### Data flow (happy path)
User message (image)
  └─> Filter: Dynamic Vision Router Max
        ├─ detect images in last user message
        ├─ side-call to Ollama VLM (bounded tokens)
        ├─ parse structured output: SUMMARY / DETAILS / OCR / UNCERTAINTY
        ├─ quality score
        ├─ optional retry (only if weak)
        ├─ optional OCR-only pass (only if user asks for text and OCR missing)
        ├─ optional verifier pass (default: only on low-quality/risky outputs)
        ├─ classify meta type: photo | document | graph
        └─ inject into context (system):
             - VISION_META
             - VISION ANALYSIS
             - VERIFIER RESULT
             - VISION_NEEDS_FOLLOWUP (YES/NO)
             - VERIFIER_MISSING (optional list)

Main text LLM (unchanged)
  └─> Produces final answer (prefers clarification over guessing)

### Optional follow-ups (pipes)
- Vision Follow-up Pipe:
    triggers when VISION_NEEDS_FOLLOWUP=YES (or missing list present),
    runs a targeted VLM query for missing elements and appends VISION_FOLLOWUP_RESULT.

- Graph Follow-up Pipe:
    triggers only when VISION_META type=graph,
    requests structure extraction (nodes, edges, labels, legend) with minimal interpretation,
    appends GRAPH_FOLLOWUP_RESULT.

### Safety rails (LowHW)
- max_concurrent_sidecalls (default 1)
- max_output_tokens cap (num_predict)
- injection trimming (max_injected_chars)
- follow-up cooldown + per-chat limit (anti-loop)

---

## Requirements
- Open WebUI with Functions enabled
- Ollama running locally or on LAN
- A vision-capable model available in Ollama (a VLM)

---

## Installation (recommended: “suite” approach)
1) **Install the Filter**:
   - Open WebUI → Admin → Functions
   - Add the Filter script: `filters/dynamic_vision_router_max.py`
   - Enable it and attach it to the model(s) you want to enhance

2) **Install optional Pipes**:
   - Add `pipes/vision_followup_pipe.py`
   - Add `pipes/graph_followup_pipe.py`
   - Pipes appear as separate selectable “models” in the UI

3) **Configure Valves**:
   - Set `ollama_base_url` (default: `http://localhost:11434`)
   - Set `vision_model_id` (your VLM in Ollama)

---

## EU language support (language packs)
This suite supports a **language preset** used for the *content* of the vision output.
Section headers remain stable for parsing (`SUMMARY/DETAILS/OCR/UNCERTAINTY`).

- `language_preset = auto` (default) tries to follow the user language
- Or force a language: `en`, `pl`, `de`, `fr`, `es`, `it`

---

## Marker contract (Filter ↔ Pipes)
The Filter injects stable metadata for optional follow-up pipes:

- `VISION_META:` includes (at minimum):
  - `- type: photo|document|graph`
- `VISION_NEEDS_FOLLOWUP: YES|NO`
- Optional: `VERIFIER_MISSING:` bullet list when something is likely missing

---

## Recommended Valve presets

### Preset A — LowHW (consumer-grade, small models)
- `max_concurrent_sidecalls = 1`
- `timeout_s = 60` (or 90 if your VLM is slow)
- `max_output_tokens = 350–450`
- `multi_image_strategy = "last"`
- `enable_vision_retry = false` (or true if you accept extra compute)
- `enable_ocr_focus_pass = true` (triggers only when user asks for text + OCR missing)
- `verifier_mode = "on_low_quality"` (or `never` for max speed)
- `max_injected_chars = 8000–12000`

### Preset B — Balanced
- `max_concurrent_sidecalls = 1`
- `timeout_s = 90`
- `max_output_tokens = 650`
- `multi_image_strategy = "last"`
- `enable_vision_retry = true`
- `enable_ocr_focus_pass = true`
- `verifier_mode = "on_low_quality"`
- `max_injected_chars = 12000`

### Preset C — MaxAccuracy (power users)
- `max_concurrent_sidecalls = 1–2` (only if your hardware can take it)
- `timeout_s = 120`
- `max_output_tokens = 900–1200`
- `multi_image_strategy = "sequential"`
- `enable_vision_retry = true`
- `enable_ocr_focus_pass = true`
- `verifier_mode = "always"`
- `max_injected_chars = 16000–24000`

---

## Support
If this suite helps you, you can support development on Ko-fi: https://ko-fi.com/cyberjaeger
