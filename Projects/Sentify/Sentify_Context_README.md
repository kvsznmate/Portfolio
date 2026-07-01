# Sentify - Emotion Classification MLOps Pipeline

> An end-to-end MLOps system that turns video or audio into per-sentence emotion predictions. Built for a TV production client who wanted to understand which parts of episodes engage viewers emotionally.

<img src="./sentify_advert_4.html" width="600" alt="Advert screen for Sentify.">

**Programme:** Applied Data Science & AI, Breda University of Applied Sciences

**Team size:** 5 students, agile Scrum, two-week sprints

**Duration:** ~8 weeks

**My role:** Scrum master

---

## The Business Problem

The client **Content Intelligence Agency**  needed a way to quantify viewer engagement at sub-episode level. They wanted to answer questions like:

- Which scenes in an episode trigger the strongest emotional response?
- How does emotional pacing differ across genres, seasons or scenes?

 They needed an automated, production-grade pipeline that could process episodes and surface emotion-level insight on demand.

## What We Built

A modular Python system that accepts a video or audio clip and returns a timestamped transcript annotated with **per-sentence emotion predictions and confidence scores** across the Ekman 7 emotions (anger, disgust, fear, happiness, neutral, sadness, surprise).

The full system is:

- **Containerised** with Docker (API + frontend + database + training)
- **Deployed three ways:** locally (`docker-compose`), on-premise (Portainer), and on Azure (Azure ML endpoints + Container Apps)
- **CI/CD'd** via GitHub Actions with a manual approval gate before production.
- **Orchestrated** in Azure ML versioned data assets, registered environments, training pipelines with hyperparameter sweeps, and conditional model registration based on evaluation metrics
- **Monitored** with both operational metrics (latency, error rate, drift) and a business-facing dashboard for stakeholders
- **Retrainable** automatically on a weekly schedule and on user-feedback triggers


![Sentify_avert_screen](sentify_advert_4.html)

### High-level pipeline

```
Video/Audio ──► Audio extraction (ffmpeg)
            ──► Speech-to-text (Whisper / AssemblyAI)
            ──► Translation to English (M2M-100, non-English audio only)
            ──► Sentence segmentation (NLTK)
            ──► Emotion classification (fine-tuned BERT)
            ──► Confidence aggregation
            ──► Timestamped predictions CSV
```

## Tech Stack

| Layer | Choice |
|---|---|
| Language | Python 3.11 |
| ML framework | PyTorch + HuggingFace Transformers |
| API | FastAPI |
| Frontend | React |
| CLI | Typer |
| Containerisation | Docker, docker-compose |
| On-prem orchestration | Portainer |
| Cloud | Azure ML (workspace, data assets, pipelines, endpoints, model registry) |
| CI/CD | GitHub Actions |
| Documentation | Sphinx |
| Testing | pytest + coverage |
| Project management | Azure DevOps Boards (Scrum) |

---

## My Contributions

The team divided work across the inference pipeline, training pipeline, deployment, monitoring, and frontend. My personal ownership covered four areas:

### 1. Multilanguage transcription pipeline

**Problem.** The initial inference pipeline only handled English audio via the AssemblyAI cloud API. The client's content portfolio spans dozens of languages, without multilingual support, the product wasn't shippable.

**Solution.** I designed and built a parallel transcription backend that runs entirely locally, supports **31 languages**, and integrates cleanly into the existing pipeline through a routing layer.


### 2. Comment-analysis pipeline

For business-side validation, I built a second pipeline that pulls viewer comments from the source video (e.g. YouTube) and runs emotion classification over the comment text. This gives the client a second signal, *audience-expressed* emotion alongside *content-carried* emotion and lets them cross-check the model's transcript-level predictions against actual viewer reaction. It also feeds the business-insight dashboard with engagement context the audio alone can't provide.

### 3. Sphinx documentation

I set up the Sphinx documentation build and authored the docstring conventions the whole team works to. Every public function and class in the package carries reStructuredText docstrings with `:param:`, `:returns:`, `:raises:`, design notes, and cross-references to related modules. 


### 4. README and developer documentation

I wrote the user-facing CLI documentation (`README_CLI.md`)  installation, quick-start examples per language, command reference, supported languages table, transcription backend selection logic, model size table, emotion label reference, and a pipeline architecture diagram. The goal was: a new user should be able to install the package and run their first inference within five minutes, without reading source code.

I also contributed to the root `README.md` (project overview, three-deployment instructions, environment variable reference) and per-component READMEs for the API and the data pipeline.


---

