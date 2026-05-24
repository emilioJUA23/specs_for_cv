# CLAUDE.md — specs_for_cv

This repo follows **spec-driven development**: every project, bullet point, or achievement on the CV starts as a written spec before any content is drafted. The goal is a CV that lands interviews for **Data Engineering**, **MLOps**, and **AI Engineering** roles.

---

## Spec-Driven Development Workflow

Each project lives in its own folder under `projects/`. The folder contains exactly two output files:

- **`spec.md`** — what the project is, why it matters, and how it maps to CV/JD requirements.
- **`implementation.md`** — a detailed implementation plan: architecture, tech stack, step-by-step build plan, and open questions.

When both files are complete and reviewed, the folder is done. The `implementation.md` is then taken to a separate repo where the actual project is built. **No coding happens in this repo.**

### Workflow

1. **Create project folder** — `projects/<slug>/`
2. **Write `spec.md`** — fill in all sections below; flag anything unknown.
3. **Write `implementation.md`** — detail architecture, components, data flow, tech choices, and build steps.
4. **Review against JD** — confirm keyword coverage and role alignment.
5. **Mark done** — only when both files are complete and metrics are confirmed (not estimated).
6. **Export** — take `implementation.md` to the build repo. This repo stays doc-only.

### `spec.md` format

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
Key tech: Spark, dbt, Airflow/Prefect, Kafka, Snowflake/BigQuery/Redshift, datalake/lakehouse, SQL/Nosql at scale, CDC, orchestration.

### MLOps
Focus on: model lifecycle, reproducibility, CI/CD for ML, monitoring, infra.
Key tech: MLflow, Kubeflow, SageMaker, Vertex AI, Docker/Kubernetes, feature stores, model serving (Triton, TorchServe, vLLM), drift detection.

### AI Engineering
Focus on: LLM integration, RAG systems, agents, prompt engineering, evaluation, inference optimization.
Key tech: LangChain/LlamaIndex, OpenAI/Anthropic APIs, vector DBs (Pinecone, Weaviate, pgvector), embeddings, fine-tuning, evals frameworks.

### `implementation.md` format

```markdown
## [Project Title] — Implementation Plan

**Architecture overview:** High-level diagram or description of components and data flow.

**Tech stack:**
| Layer | Choice | Reason |
|-------|--------|--------|
| ...   | ...    | ...    |

**Components:**
- Component A — what it does
- Component B — what it does

**Build steps:**
1. Step one
2. Step two
...

**Open questions / decisions deferred:**
- Question or risk that needs resolution before/during build
```

---

## Rules for Claude

- This repo is **documentation only** — never write code, scripts, or runnable files.
- Always start by reading the relevant spec before writing or editing any CV content.
- Never invent metrics — if impact is unknown, flag it and ask the user.
- Bullet points must follow the **action → tool/method → quantified result** pattern.
- Keep bullets under 2 lines. No fluff, no passive voice.
- When reviewing specs, flag missing impact data and missing JD keyword coverage.
- When multiple roles are targeted, create role-specific variants of bullets if framing differs.
- Do not reformat the entire CV because one section changed — surgical edits only.

---

## Agent Roles (Slash Commands)

Spec and implementation work is driven through a set of role-based slash commands. Each command puts Claude in a specific expert persona that asks questions, challenges assumptions, and researches the web when needed. The user acts as the **Infrastructure Architect** — these roles fill the surrounding expertise gaps.

| Command | Role | Use it when… |
|---|---|---|
| `/role-senior-data-engineer <topic>` | Senior Data Engineer | Designing pipelines, storage layers, orchestration, data modeling, CDC, dbt |
| `/role-ml-domain-expert <topic>` | Senior ML/Data Domain Expert | ML design, feature engineering, model selection, RAG architecture, evals |
| `/role-tech-researcher <topic>` | Principal Tech Researcher | Comparing frameworks, finding benchmarks, validating tech choices with evidence |
| `/role-jd-analyst <topic>` | JD & Market Analyst | Keyword coverage, CV bullet quality, differentiation, role-target alignment |
| `/role-devil-advocate <topic>` | Scope Critic | Finding spec weaknesses, pressure-testing impact claims, scoping down when needed |

Skills are defined in `.claude/commands/`. Each skill leads with questions before answers and uses WebSearch to back claims with current data.

---

## Repo Structure

```
specs_for_cv/
├── CLAUDE.md                        # this file
├── .claude/
│   └── commands/                    # role-based slash commands (agent personas)
├── projects/
│   └── <slug>/
│       ├── spec.md                  # what the project is + CV framing
│       └── implementation.md        # architecture + build plan (exported to build repo)
├── cv/
│   ├── base.md                      # master CV in markdown
│   ├── data_engineering.md
│   ├── mlops.md
│   └── ai_engineering.md
└── jd_analysis/                     # paste JDs here for keyword extraction
```

---

## Definition of Done (per project folder)

- [ ] `spec.md` written and reviewed
- [ ] `implementation.md` written with full build plan
- [ ] Impact metrics confirmed (not estimated)
- [ ] JD keyword coverage checked
- [ ] CV bullet(s) drafted from spec
- [ ] Bullet added to the correct role variant(s) of the CV
- [ ] `implementation.md` exported to build repo
