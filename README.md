# Promo Extractor — Kuku AI Generalist Assignment, Problem 1

Reads a microdrama screenplay and finds 20–60 second promo clips that open on a hook
(first 3 seconds), cut on a cliffhanger, and make sense to someone who has never seen
the show. Outputs every promo's episode, start/end timestamps, hook, cliffhanger, a
one-line reason, a tier, and a posting order.

## How it works

| Phase | Who | What |
|---|---|---|
| 1 Parse | Python | `.txt` script → timeline of camera / action / dialogue / SFX events, times in seconds, shot cut points |
| 2 Hook Hunter | LLM, 1 call per episode | reads the whole episode, returns 3–5 hook start times |
| 3 Framer | Python | hook time T → window T..T+60s (never past the episode end); rejects timestamps that match no line |
| 4 Judge & Trimmer | LLM, 1 call per clip | picks the cut (20–60s), scores hook / cliffhanger / middle, runs the amnesia test, writes hook, cliffhanger, why |
| 4b Timestamp check | Python | snaps the chosen end to a real line end or shot cut inside 20–60s; logs every adjustment |
| 5 Sorter + Ranker | Python + LLM, 1 call | Tier 1 ready / Tier 2 editor splice / Tier 3 bad picks; LLM orders Tier 1 for posting |
| 5b Hand check | Human | labels the top picks blind; `--score` reports how often the tool was right |

The model never owns time: every timestamp it returns is checked against the script,
and clip length is enforced in code.

## Run it

**Colab (recommended):** open `notebook.ipynb` in Colab, add `OPENAI_API_KEY` in the
Secrets panel, upload the scripts zip, run the cells in order.

**Locally:**
```bash
pip install -r requirements.txt
export OPENAI_API_KEY=...
python tests/test_checks.py                       # 16 checks, no API calls
python run.py scripts.zip --provider mock         # offline dry run
python run.py scripts.zip --episodes 1-3          # quick real run
python run.py scripts.zip --out outputs           # full show
python run.py --score outputs/hand_check.csv      # after labelling the sheet
streamlit run app.py                              # dashboard
```

## Swap the model

Edit the `llm:` block in `config.yaml`, or pass `--model` / `--provider` / `--fallback`:
- `provider: openai` (default), `model: gpt-5-mini`, `fallback_models: [gpt-4o-mini]`; answers
  use OpenAI's strict structured outputs, so they always match the schema
- `provider: anthropic`, `model: claude-sonnet-5-5`, `fallback_models: [claude-haiku-5-5]`
- `provider: gemini`, `model:` any Gemini model ID your key can use
- `provider: openai_compatible` with `base_url` + `api_key_env` → Groq, OpenRouter,
  OpenAI, Ollama, or any API that speaks the OpenAI chat format
- `provider: mock` → offline keyword heuristics, for testing only

## Outputs (`outputs/`)

| File | What |
|---|---|
| `shortlist.md` | Tier 1 in posting order, then Tier 2 and Tier 3 with reasons |
| `promos.json` | every judged clip with segments, scores, notes, and the run metadata |
| `promos.csv` | the same, one row per clip |
| `failures.csv` | rejected hooks, Tier 3 reasons, adjusted timestamps |
| `hand_check.csv` | top picks without model scores, for blind human labelling |
| `run_log.json` | counts, API calls, cache hits, tokens, time taken, config used |

## Cost and scale

One show of 50 episodes ≈ 50 Hook Hunter calls + up to 250 Judge calls + 1 Ranker call:
roughly 0.75M input and 0.12M output tokens. `run_log.json` records the exact token counts,
so cost per show = tokens × the model's price. Calls are paced by `requests_per_minute`
in `config.yaml`, and every response is cached in `.cache/`, so re-runs and interrupted
runs don't repeat calls. If the main model is overloaded or rate-limited, the backup in
`fallback_models` takes over; if credits run out, the run stops and saves what it has.

## Not used in the submitted run

`rule_maker.py` + `reference_promos.txt` can learn hook and cliffhanger patterns from 60
ReelShort / DramaBox promo Shorts (Gemini watches the YouTube links) and write `rules.json`,
which `run.py` then adds to the prompts. It was left out of the submitted run, which grades
on the brief's definition of a good promo.

## Known limits

- Works from the script, not the video: it can't judge acting, pacing on screen or audio.
- One continuous segment per promo; it doesn't stitch moments across episodes yet.
- Clips never cross episode boundaries; a hook in the last 20s of an episode is dropped.
- The amnesia test and scores are the model's judgement, so check them with the hand-check sheet.

## Confidentiality

Kuku's scripts are confidential. `.gitignore` keeps scripts, outputs and the cache out of
the repo; upload the zip at run time instead.
