# CLAUDE.md — specs_for_cv

This repo follows **spec-driven development**: every project, bullet point, or achievement on the CV starts as a written spec before any content is drafted. The goal is a CV that lands interviews for **Data Engineering**, **MLOps**, and **AI Engineering** roles.

---

## Spec-Driven Development Workflow

1. **Spec first** — before writing any CV content, create a spec file under `specs/` describing what the project/achievement is, its impact, and which target role(s) it serves.
2. **Draft from spec** — CV content is generated strictly from the approved spec. No improvisation.
3. **Review against JD** — each spec should map to real job description keywords and requirements.
4. **Iterate** — update the spec when scope or framing changes; re-derive CV content from the updated spec.

### Spec file format (`specs/<slug>.md`)

```markdown
## [Project / Achievement Title]

**Target roles:** Data Engineering | MLOps | AI Engineering (pick relevant)

**One-liner:** < 20 words summarizing what you built or did >

**Context:** What problem existed before this? What was the scale?

**What you did:** Concrete actions, tools, technologies used.

**Impact:** Quantified outcomes (latency, cost, accuracy, throughput, uptime…).

**Keywords:** comma-separated JD keywords this covers

**CV bullet(s):**
- Draft bullet point(s) derived from the above
```

---

## Target Role Profiles

### Data Engineering
Focus on: pipeline reliability, scalability, data quality, cost efficiency.
Key tech: Spark, dbt, Airflow/Prefect, Kafka, Snowflake/BigQuery/Redshift, datalake/lakehouse, SQL at scale, CDC, orchestration.

### MLOps
Focus on: model lifecycle, reproducibility, CI/CD for ML, monitoring, infra.
Key tech: MLflow, Kubeflow, SageMaker, Vertex AI, Docker/Kubernetes, feature stores, model serving (Triton, TorchServe, vLLM), drift detection.

### AI Engineering
Focus on: LLM integration, RAG systems, agents, prompt engineering, evaluation, inference optimization.
Key tech: LangChain/LlamaIndex, OpenAI/Anthropic APIs, vector DBs (Pinecone, Weaviate, pgvector), embeddings, fine-tuning, evals frameworks.

---

## Rules for Claude

- Always start by reading the relevant spec before writing or editing any CV content.
- Never invent metrics — if impact is unknown, flag it and ask the user.
- Bullet points must follow the **action → tool/method → quantified result** pattern.
- Keep bullets under 2 lines. No fluff, no passive voice.
- When reviewing specs, flag missing impact data and missing JD keyword coverage.
- When multiple roles are targeted, create role-specific variants of bullets if framing differs.
- Do not reformat the entire CV because one section changed — surgical edits only.

---

## Repo Structure

```
specs_for_cv/
├── CLAUDE.md           # this file
├── specs/              # one .md per project/achievement
├── cv/
│   ├── base.md         # master CV in markdown
│   ├── data_engineering.md
│   ├── mlops.md
│   └── ai_engineering.md
└── jd_analysis/        # paste JDs here for keyword extraction
```

---

## Definition of Done (per spec)

- [ ] Spec written and reviewed
- [ ] Impact metrics confirmed (not estimated)
- [ ] JD keyword coverage checked
- [ ] CV bullet(s) drafted from spec
- [ ] Bullet added to the correct role variant(s) of the CV
