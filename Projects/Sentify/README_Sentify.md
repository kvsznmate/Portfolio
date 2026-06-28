# Emotion MLOps — Backend

NLP emotion-classification service for Banijay Benelux. Given a video or audio clip, it returns a transcript with per-sentence emotion predictions (Ekman 6 + neutral) and confidence scores.

This is the Python backend: the installable package, CLI, FastAPI service, and training / retraining code. The React frontend lives in `../frontend/`.

Part of the BUas Block D MLOps project. See the root [README](../README.md) for team and project context.

---

## Prerequisites

- **Python 3.12+** — if you don't have it: <https://www.python.org>
- **uv** — Python package manager. Install with:
  ```bash
  pip install uv
  ```
  Or via [Astral's installer](https://docs.astral.sh/uv/getting-started/installation/) for a system-wide install that doesn't depend on an existing Python.
- **The trained model checkpoint** — not in git (too large, `*.pth` gitignored). Two options:
  - Hand-place it: ask the team for `final_bert_model.pth` and place it at `backend/ml_models/final_bert_model.pth`. The API and CLI default to this path.
  - Fetch from Azure ML registry at startup: set `INFERENCE_MODEL_SOURCE=azureml` plus the related `INFERENCE_*` and `AZURE_*` env vars (see [Model loading](#model-loading) below). The container fetches and caches the model on startup.
- **AssemblyAI API key** — for English transcription. Create `backend/.env` with:
  ```
  WHISPER_API=your_assemblyai_key_here
  ```
  `.env` is gitignored. Ask the team for a shared key if you don't have one.

---

## Install

From `backend/`:

```bash
uv sync --extra dev
```

This creates `.venv/`, installs all project dependencies, and adds the dev tools (pytest, pytest-cov, pytest-mock, ruff). The `--extra dev` flag is required for running tests or linting — without it you only get runtime dependencies.

Verify the install:

```bash
uv run emotion-mlops --help
```

You should see Typer's help screen. If you get `command not found`, the console script didn't register — run `uv sync --extra dev` again.

---

## Usage

### As a CLI

The CLI exposes commands at two levels:

- **Top-level user commands:** `run`, `download`, `transcribe`, `predict`, `train`, `tune` — for end-to-end inference, training, and hyperparameter tuning.
- **Stage commands:** `stages preprocess`, `stages train`, `stages evaluate` — individual training pipeline stages, used by automation (Azure ML pipeline components invoke these one at a time). Not typical for direct human use.

**Full inference pipeline** (download → transcribe → predict):

```bash
uv run emotion-mlops run "https://www.youtube.com/watch?v=<id>"
```

**Individual inference steps:**

```bash
# Download audio only
uv run emotion-mlops download "https://www.youtube.com/watch?v=<id>"

# Transcribe an existing audio file
uv run emotion-mlops transcribe data/audio/episode.m4a

# Predict emotions on an existing transcript CSV
uv run emotion-mlops predict data/transcripts/episode_transcript.csv
```

**Train a new model end-to-end:**

```bash
uv run emotion-mlops train --config config.yaml
```

All training hyperparameters (data paths, model, epochs, batch size, output directory) come from the YAML config — see `src/emotion_mlops/core/config_reader.py` for the schema. Outputs (checkpoint + training curves + confusion matrix) land in the directory specified by `output.dir` in the config.

**Run hyperparameter tuning:**

```bash
uv run emotion-mlops tune \
  --config config.yaml \
  --hpo-config hpo_config.yaml \
  --processed-dir data/processed \
  --output-dir hpo_runs/study-01
```

Runs an Optuna study over the search space defined in the HPO config. Each trial runs the train stage with sampled hparams; the runner writes a `study_summary.json` with the winning trial's full provenance. Used by the Azure ML HPO pipeline.

**Individual training pipeline stages:**

```bash
# Preprocess raw CSV into train/val/test parquet splits
uv run emotion-mlops stages preprocess \
  --in-csv data/raw/emotions.csv \
  --out-dir data/processed \
  --config config.yaml

# Train against existing processed splits
uv run emotion-mlops stages train \
  --processed-dir data/processed \
  --model-out runs/exp-01 \
  --config config.yaml

# Evaluate an existing checkpoint against the test split
uv run emotion-mlops stages evaluate \
  --model-path runs/exp-01/final_bert_model.pth \
  --processed-dir data/processed \
  --metrics-out runs/exp-01/metrics \
  --config config.yaml
```

The `stages train` command also accepts `--init-from-checkpoint <path>` for warm-starting from an existing checkpoint. This is the entry point used by the Azure ML retraining (fine-tune) pipeline; when omitted, training proceeds from `bert-base-uncased` as usual.

The stage commands also accept `--override key=value` (repeatable) to override individual config fields at runtime without editing the YAML — the Azure ML pipeline uses this to inject HPO-discovered hyperparameters.

**See all commands and options:**

```bash
uv run emotion-mlops --help
uv run emotion-mlops <command> --help
uv run emotion-mlops stages --help
```

Each subcommand prints its output path on success. Output of `run` lands in `backend/data/predictions/episode_final.csv` with columns: `Start Time`, `End Time`, `Sentence`, `Emotion`, `Confidence`. Intermediate files land in `backend/data/audio/` and `backend/data/transcripts/`. All of `backend/data/` is gitignored.

### As a library

```python
from emotion_mlops.services.prediction import predict_emotions, EMOTIONS
from emotion_mlops.services.transcription import transcribe_audio, save_sentences_to_csv
from emotion_mlops.services.downloader import download_youtube_audio

# EMOTIONS = ['anger', 'disgust', 'fear', 'happiness', 'neutral', 'sadness', 'surprise']
```

---

### As an HTTP API

Start the dev server:

    uv run uvicorn emotion_mlops.api.main:app --reload --host 0.0.0.0 --port 8000

Then:

- **Swagger docs:** <http://localhost:8000/docs> — interactive API explorer
- **OpenAPI schema:** <http://localhost:8000/openapi.json>
- **Health check:** `curl http://localhost:8000/health` → `{"status": "ok"}`
- **Metadata:** `curl http://localhost:8000/metadata` → reports which model the container is serving (source, name, version, path, image tag, git SHA). Used by deployment smoke tests.
- **Predict:** POST a JSON body with a YouTube URL:

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.youtube.com/watch?v=<id>"}'
```

Response structure:

```json
{
  "url": "https://www.youtube.com/watch?v=<id>",
  "total_sentences": 34,
  "sentences": [
    {
      "start_time": "00:00:00",
      "end_time": "00:00:02",
      "sentence": "Hello world.",
      "emotion": "happiness",
      "confidence": 0.87
    }
  ]
}
```

**Note:** `/predict` is synchronous and blocks for 30+ seconds per request while the pipeline runs. Set a generous client timeout. Async job-based processing (with a `/jobs/{id}` polling endpoint) may be added later — see tech debt note in `emotion_mlops/api/main.py`.

### Model loading

The FastAPI container resolves which checkpoint to serve at startup, via `emotion_mlops.services.model_loader.fetch_model_from_registry()`. Two sources are supported, selected by `INFERENCE_MODEL_SOURCE`:

| Source | Behavior |
|---|---|
| `local` (default) | Use a checkpoint already on disk. Path is `INFERENCE_MODEL_LOCAL_PATH` if set, otherwise `core.config.MODEL_PATH` (`backend/ml_models/final_bert_model.pth`). |
| `azureml` | Fetch a registered model from the Azure ML registry on startup. Cache it on a host-mounted volume so subsequent restarts skip the download. |

When the registry fetch fails for any reason (auth error, missing model, network), the API logs the error and falls back to `core.config.MODEL_PATH`. This means a Portainer stack configured for `source=local` keeps working even when Azure is unreachable.

Environment variables read by the model loader:

| Variable | Required | Description |
|---|---|---|
| `INFERENCE_MODEL_SOURCE` | No (default `local`) | `"azureml"` for production fetch, `"local"` for hand-placed checkpoint. |
| `INFERENCE_MODEL_NAME` | When `source=azureml` | Registered model name (e.g. `emotion-bert`). |
| `INFERENCE_MODEL_VERSION` | When `source=azureml` | Version string. `"latest"` is supported. |
| `INFERENCE_MODEL_CACHE_DIR` | When `source=azureml` | Host path for the on-disk cache (per-version subdirectory). |
| `INFERENCE_MODEL_LOCAL_PATH` | No | Direct path to a `.pth` file. Overrides the default `MODEL_PATH`. |
| `INFERENCE_MODEL_CHECKPOINT_FILENAME` | No (default `final_bert_model.pth`) | Override if the registered bundle uses a different filename. |

Azure SDK credentials (only needed when `source=azureml`):

| Variable | Description |
|---|---|
| `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET` | Service principal credentials. |
| `AZUREML_SUBSCRIPTION_ID`, `AZUREML_RESOURCE_GROUP`, `AZUREML_WORKSPACE_NAME` | Workspace identifiers. |

For details on how the Azure ML registry is populated by the training, HPO, and retraining pipelines, see [`azure/azure_ml_workflow.md`](azure/azure_ml_workflow.md).

## Development

### Project structure

```
backend/
├── src/emotion_mlops/         # the installable package
│   ├── cli.py                 # Typer CLI entry point (top-level + stages subcommands)
│   ├── api/                   # FastAPI app
│   │   └── main.py            # /predict, /health, /metadata, lifespan model fetch
│   ├── core/                  # config, logging
│   │   ├── config.py          # paths, API keys, device
│   │   └── config_reader.py   # YAML config parsing for training
│   ├── pipelines/             # Azure ML pipeline factories (Python @pipeline)
│   │   ├── training_pipeline.py
│   │   └── retraining_pipeline.py
│   ├── azureml_components/    # Python entry points for Azure ML pipeline components
│   │   ├── hparams_adapter.py
│   │   ├── bundle.py
│   │   └── select_winner.py
│   ├── registration/          # Conditional model registration
│   │   └── conditional_register.py
│   └── services/              # pipeline stages and runtime services
│       ├── downloader.py      # yt-dlp audio extraction
│       ├── transcription.py   # AssemblyAI + sentence segmentation
│       ├── prediction.py      # BERT emotion classification
│       ├── model_loader.py    # registry / local checkpoint resolution
│       └── training/          # training subpackage
│           ├── data.py        # text cleaning, dataset, dataloaders
│           ├── loop.py        # train_one_epoch, evaluate
│           ├── checkpoint.py  # bundle save/load with arch checks
│           ├── plots.py       # training curves + confusion matrix
│           ├── runner.py      # train(cfg) -> TrainingResult entrypoint
│           └── stages/        # preprocess / train / evaluate stage entry points
├── azure/                     # Azure ML infrastructure
│   ├── components/            # Component YAMLs
│   ├── envs/                  # Environment definition
│   ├── jobs/                  # Smoke job
│   ├── scripts/               # Submitters and infrastructure scripts
│   └── azure_ml_workflow.md   # Architecture reference
├── tests/                     # pytest suite, mirrors src/ layout
│   ├── conftest.py            # shared fixtures
│   └── unit/
├── data/                      # runtime outputs (gitignored)
├── ml_models/                 # model checkpoints (gitignored)
├── pyproject.toml             # project metadata, deps, pytest + coverage config
└── uv.lock                    # resolved dependency versions
```

Runtime data and model checkpoints live at `backend/data/` and `backend/ml_models/` — outside the package. This is deliberate: the installed package should be immutable at runtime, with data paths driven by config.

### Running tests

```bash
uv run pytest                                   # all tests
uv run pytest tests/unit/services               # one directory
uv run pytest -k "config_reader"                # tests matching a pattern
uv run pytest --cov --cov-report=term-missing   # with coverage
uv run pytest --cov --cov-report=html           # HTML report in htmlcov/
```

Current suite: approximately 320 tests, ~10s runtime on a clean uv venv.

### Linting and formatting

```bash
uv run ruff check .
uv run ruff format .
```

`ruff` is configured in `pyproject.toml`. Pre-commit hooks are wired via `.pre-commit-config.yaml`.

### Adding a dependency

```bash
uv add <package>              # runtime dependency
uv add --dev <package>        # dev-only (test/lint tooling)
```

Commit both `pyproject.toml` and `uv.lock` together. Don't edit `uv.lock` by hand.

---

## Building the wheel

The backend ships as an installable Python package (`.whl` + `sdist`):

```bash
uv build
```

Output goes to `dist/`:

```
dist/
├── emotion_mlops-0.1.0-py3-none-any.whl
└── emotion_mlops-0.1.0.tar.gz
```

To verify the wheel installs cleanly in a fresh environment:

```bash
uv venv .venv-test
.venv-test/Scripts/activate          # Windows
# source .venv-test/bin/activate     # macOS/Linux
pip install dist/emotion_mlops-0.1.0-py3-none-any.whl
emotion-mlops --help
deactivate
rm -rf .venv-test
```

---

## Troubleshooting

**`command not found: emotion-mlops`** — The console script isn't installed. Run `uv sync --extra dev`.

**`emotion-mlops <url>` gives a "No such command" error** — The CLI requires a subcommand. Use `emotion-mlops run <url>` instead.

**`FileNotFoundError: .../ml_models/final_bert_model.pth`** — The checkpoint isn't in `backend/ml_models/`. Either ask the team for it, or configure the registry fetch via `INFERENCE_MODEL_SOURCE=azureml` and the related env vars.

**`ModuleNotFoundError: No module named 'typer'`** (or `pytest_mock`, etc.) — Your environment is out of sync with `pyproject.toml`. Run `uv sync --extra dev`.

**AssemblyAI errors** — Check `backend/.env` contains `WHISPER_API=...` with a valid key. The key is loaded at import time by `python-dotenv`.

**`uv: command not found`** — uv isn't installed. Run `pip install uv` or follow the [Astral installer](https://docs.astral.sh/uv/getting-started/installation/).

**Tests hang or take minutes when the suite normally runs in 10s** — Usually means `MLFLOW_TRACKING_URI` is set in your shell to a server that isn't reachable. The test suite's autouse fixture strips it; if a test bypasses that or the fixture is removed, `mlflow.set_experiment` retries with exponential backoff. Unset the var or start the local MLflow service.

**Tests pass locally but fail in CI** — Usually means a dependency is used but not declared in `pyproject.toml`. Run `uv sync --extra dev` from a fresh clone to reproduce.

---

## Deliverable status

| Deliverable | Status | Evidence |
|---|---|---|
| Installable Python package | Done | `uv build` produces `.whl` + `sdist` |
| Console script (CLI) | Done | `emotion-mlops` exposes `run`, `train`, `tune`, `stages preprocess|train|evaluate`, etc. |
| Unit tests | Done | ~320 tests in `tests/unit/` |
| Test coverage | Done | High coverage on in-scope modules; thresholds in `pyproject.toml` |
| Logging | Done | `core/logging.py`, used across services |
| Docstrings | Done | Google-style across public functions |
| Error handling | Done | CLI exits non-zero on pipeline failure; transcription and registration raise with descriptive errors |
| HTTP API | Done | `POST /predict`, `GET /health`, `GET /metadata`, Swagger docs at `/docs` |
| Linting | Done | `ruff check` and `ruff format` configured; pre-commit hooks wired |
| Sphinx docs | Planned | Docstrings in place; Sphinx config not yet set up |
| Async job endpoint (`POST /jobs`, `GET /jobs/{id}`) | Planned | Current `/predict` is synchronous |

---

## Related references

- [`azure/azure_ml_workflow.md`](azure/azure_ml_workflow.md) — End-to-end architecture of the Azure ML side: data assets, three pipelines (HPO, training, retraining), conditional registration, deployment.
- [`../frontend/`](../frontend/) — React frontend.
