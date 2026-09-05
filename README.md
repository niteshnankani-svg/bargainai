# BargainAI

> India's first Hinglish negotiation agent — MuRIL embeddings + hierarchical RAG + Anthropic Claude, deployed as a Gradio agent (HuggingFace Spaces) with a WhatsApp bridge and a Shopify storefront app.

This repo consolidates the BargainAI implementations that were previously split across separate repos. Each lives in its own subdirectory.

## Problem statement

Indian e-commerce and marketplace buyers naturally negotiate in Hinglish (mixed Hindi-English), but existing AI assistants are English-only and culturally tone-deaf to negotiation dynamics. BargainAI is a seller-side assistant that understands Hinglish context and responds with culturally appropriate counter-offers.

## Variants in this repo

| Variant | Path | What it is |
|---|---|---|
| Core negotiation agent + WhatsApp bridge | [`webhook/`](webhook/) | The Gradio negotiation agent (`app.py`), plus a small Flask webhook (`whatsapp_webhook.py`) that relays Twilio WhatsApp messages to it |
| Shopify app | [`shopify/`](shopify/) | FastAPI app that installs into a Shopify store via OAuth and runs negotiation logic at checkout |

A static product walkthrough is also included: [`bargainai_presentation.html`](bargainai_presentation.html).

## Architecture

**Core agent (`webhook/app.py`)** — a Gradio app. User messages (Hinglish or English) are embedded using MuRIL (Google's multilingual BERT for Indian languages) or BERT depending on market. Precomputed embeddings (`index/muril_finetuned_embeddings.npy` + metadata, not included in this repo) provide retrieval context. Anthropic Claude generates the negotiation response conditioned on retrieved context and conversation history. This is the same logic deployed live at [huggingface.co/spaces/nitz0219/BargainAI](https://huggingface.co/spaces/nitz0219/BargainAI).

**WhatsApp bridge (`webhook/whatsapp_webhook.py`)** — a small Flask app. Receives inbound WhatsApp messages via a Twilio webhook, calls the deployed HuggingFace Space through `gradio_client` (currently hardcoded to the `nitz0219/BargainAI` space), and replies over WhatsApp via Twilio's `MessagingResponse`.

**Shopify app (`shopify/main.py`)** — FastAPI. Handles Shopify OAuth app installation, stores per-store tokens locally, and calls Anthropic Claude directly for negotiation logic at checkout.

## Tech stack

`Python` · `Gradio` · `Flask` · `FastAPI` · `MuRIL (Google)` · `Anthropic Claude` · `Twilio` · `Shopify Admin API` · `HuggingFace Transformers` · `HuggingFace Spaces`

## Setup / run

```bash
git clone https://github.com/niteshnankani-svg/bargainai
cd bargainai

# Core agent — needs model checkpoints + embeddings that aren't in this repo
# (see Known limitations); this is what's deployed on HF Spaces.
cd webhook
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # add ANTHROPIC_API_KEY
python app.py

# WhatsApp bridge — talks to the already-deployed HF Space, doesn't need
# the model files locally. Needs HF_TOKEN and Twilio configured on the
# Twilio console to point at wherever this is hosted.
cp .env.example .env   # add HF_TOKEN
python whatsapp_webhook.py

# Shopify app (separate terminal / venv)
cd ../shopify
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # add SHOPIFY_CLIENT_ID, SHOPIFY_CLIENT_SECRET, ANTHROPIC_API_KEY, SECRET_KEY
uvicorn main:app --reload
```

## Evaluation results

No eval scripts or benchmark output are committed to this repo. Latency and accuracy figures mentioned in earlier drafts of this README (sub-2s response time, 14K indexed reviews) aren't independently verifiable from what's checked in here — treat them as unverified until a real eval run is added.

## Known limitations

- `webhook/app.py` expects local model checkpoints (`models/muril_finetuned`, `models/bert_finetuned`) and precomputed embeddings (`index/*.npy`) that are not tracked in this repo — it won't run standalone without them. The live version is the deployed HF Space.
- `webhook/whatsapp_webhook.py` hardcodes the target HuggingFace Space name (`nitz0219/BargainAI`) rather than reading it from an environment variable.
- The Shopify app's OAuth flow sets a `state` parameter but never verifies it on callback — the CSRF protection this parameter is meant to provide is currently a no-op.
- No automated tests in either variant.
- No committed evaluation harness for negotiation quality or latency.
