# Autonomous HR Agent 🤖📋

![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![LangChain](https://img.shields.io/badge/LangChain-Agents-1C3C3C)
![LLM](https://img.shields.io/badge/LLM-Ollama%20%7C%20Groq-orange)
![Status](https://img.shields.io/badge/Status-Work%20in%20Progress-yellow)
![License](https://img.shields.io/badge/License-MIT-green)

An autonomous, multi-agent recruitment system built with **LangChain** that reads resumes, scores candidates against a job description, and makes independent **ADVANCE / MAYBE / REJECT** hiring decisions — complete with reasoning, strengths/concerns, and interview focus areas. No human reviews any individual step.

> Built as a hands-on exploration of autonomous agent design: chaining together perception (resume parsing), judgment (LLM-based scoring), and decision logic (thresholded hiring calls) into a single pipeline — with graceful fallbacks when the LLM misbehaves.

**TL;DR**: drop resumes + a job description into `data/`, run the notebook, get a ranked shortlist with reasoning in a few minutes instead of hours.

---

## 📑 Table of Contents

- [Why this exists](#-why-this-exists)
- [Demo output](#-demo-output)
- [How it works](#-how-it-works)
- [Architecture](#️-architecture)
- [Tech stack](#️-tech-stack)
- [Repo structure](#-repo-structure)
- [Getting started](#-getting-started)
- [Configuration](#-configuration)
- [Sample output](#-sample-output)
- [Performance metrics](#-performance-metrics-the-agent-tracks-on-itself)
- [Project status & roadmap](#-project-status--roadmap)
- [FAQ](#-faq)
- [Limitations](#️-notes--limitations)
- [Contributing](#-contributing)
- [License](#-license)

---

## 💡 Why this exists

Screening resumes is repetitive and time-consuming. HR teams routinely spend significant hours per hire on manual tasks — reading resumes, comparing them to a job description, and deciding who moves forward.

This project explores how far you can push **autonomy** in that pipeline: given a job description and a folder of resumes, can a set of LLM-driven agents independently extract candidate profiles, score them consistently, and generate a ranked shortlist — end to end, without a human reviewing each step?

## 🎬 Demo output

```
🥇 1. Jane Doe - 8.7/10
   🚀 Status: ADVANCE (High Priority)
   Action: Schedule technical interview
   Breakdown: Tech:9.1 | Exp:8.5 | Edu:8.0 | Fit:8.8
   Key Strengths: Strong Python background, relevant ML experience
   Interview Focus: System design, production ML experience

🤔 2. John Smith - 6.2/10
   Status: MAYBE (Medium Priority)
   Action: Phone screening required
   Concerns: Limited leadership experience, gap in recent employment

⏱️ PERFORMANCE METRICS - AUTONOMOUS PROCESSING
   Manual Decision Time: 15 min/candidate
   Automated Decision Time: 0.41 min/candidate
   Efficiency Improvement: 97.3%
   Cost Savings: $233.20 for 10 candidates
```

## 🧠 How it works

The system is organized into two cooperating agents that pass structured data between them — each one independently responsible for its slice of the decision, and each one able to explain *why* it decided what it decided.

### 1. `ResumeIntelligenceAgent` — perception layer

Turns unstructured resume text into structured, comparable data.

- Takes raw resume text (Markdown) as input
- Uses a structured LLM prompt (`langchain_core.prompts.PromptTemplate`) to extract:
  - Full name
  - Years of relevant experience
  - Current/most recent job title
  - Key technical skills (as a list)
  - Highest education + school
  - A 2–3 sentence professional summary
- Forces strict JSON-only output from the LLM (no preamble, no markdown fences) so downstream code can parse it reliably
- Tracks per-candidate processing time and running analysis counts

```python
agent = ResumeIntelligenceAgent(llm)
info = agent.extract_resume_info(resume_text, candidate_name="Jane Doe")
# → {"name": "Jane Doe", "experience_years": 6, "key_skills": [...], ...}
```

### 2. `DecisionEngineAgent` — judgment + decision layer

Takes the structured candidate profiles + the job description and turns them into an actual hiring call.

- Scores each candidate on **4 weighted dimensions** (0–10 scale each, combined into a `total_score`):
  - Technical skills
  - Experience
  - Education
  - Overall fit for the role
- Applies **configurable thresholds** (`advance_threshold`, `maybe_threshold`) to autonomously classify each candidate:

  | Decision | Meaning | Action |
  |---|---|---|
  | 🚀 `ADVANCE` | Score ≥ advance threshold | Schedule technical interview |
  | 🤔 `MAYBE` | Score ≥ maybe threshold | Phone screening required |
  | ❌ `REJECT` | Below maybe threshold | Send rejection email |

- Every decision comes with **reasoning**, **strengths**, **concerns**, and **suggested interview focus areas** — not just a number
- **Fails gracefully**: if the LLM doesn't return valid JSON, a keyword/sentiment-based `_calculate_fallback_score()` kicks in so the pipeline never crashes on a bad response; if *that* fails too, an emergency `MAYBE` decision is issued and flagged for manual review
- Tracks its own performance: total processing time, per-candidate timing, decision distribution, and computes estimated **time and cost savings** vs. a manual 15-min/candidate baseline

```python
engine = DecisionEngineAgent(llm, job_description)
decisions = engine.process_all_candidates(processed_resumes)
summary = engine.get_decision_summary()
# → {"total_candidates": 10, "average_score": 6.8, "decision_breakdown": {...}, ...}
```

Both agents log their own stats, and results are exported to JSON at each stage (`processed_resumes.json`, `candidate_decisions.json`, `decision_summary.json`) so later stages — or a fresh notebook session — can pick up where earlier ones left off without re-running everything.

## 🏗️ Architecture

```
data/
├── job_description.markdown
├── resumes/
│   └── *.markdown                  # one resume per candidate
└── templates/
    └── *.md                        # (for outbound communication templates)

           ┌──────────────────────────┐
resumes ──▶│  ResumeIntelligenceAgent │──▶ processed_resumes.json
           └──────────────────────────┘
                        │
                        ▼
           ┌──────────────────────────┐
job desc ─▶│    DecisionEngineAgent   │──▶ candidate_decisions.json
           └──────────────────────────┘        + decision_summary.json
                        │
                        ▼
              Ranked candidate report
           (top candidates, ADVANCE/MAYBE/REJECT queues)
```

## 🛠️ Tech Stack

| Component | Tool |
|---|---|
| Agent orchestration | [LangChain](https://python.langchain.com/) (`langchain_core`, prompts, runnables) |
| LLM (local, free) | [Ollama](https://ollama.com/) running `llama3.2` |
| LLM (hosted, optional) | [Groq](https://groq.com/) running `llama-3.3-70b-versatile` |
| Language | Python 3 |
| Data | Markdown resumes + job description |

The notebook supports **dual LLM strategy** — run entirely free/local via Ollama, or swap in Groq's hosted API for faster/larger-model inference.

## 📂 Repo structure

```
.
├── Suffian_CO_Autonomous_HR_Agent.ipynb   # main notebook (agents + pipeline)
├── data/
│   ├── job_description.markdown
│   ├── resumes/
│   └── templates/
├── processed_resumes.json                 # generated after Part 1
├── candidate_decisions.json               # generated after Part 3
├── decision_summary.json                  # generated after Part 3
├── .env.example
└── README.md
```

## 🚀 Getting started

### 1. Clone and install dependencies

```bash
git clone https://github.com/<your-username>/autonomous-hr-agent.git
cd autonomous-hr-agent
pip install -r requirements.txt
```

**requirements.txt** (core packages used):
```
langchain-core
langchain-ollama
langchain-groq
langchain-text-splitters
langchain-community
langchain-huggingface
python-dotenv
numpy
sentence-transformers
faiss-cpu
```

### 2. Set up your LLM

**Option A — Ollama (free, local, default):**
```bash
ollama serve
ollama pull llama3.2
```

**Option B — Groq (hosted):**
Create a `.env` file:
```
GROQ_API_KEY=your_key_here
```
Then in the notebook, set `setup_llm(use_groq=True)`.

### 3. Add your data

Place your files under `data/`:
```
data/
├── job_description.markdown
└── resumes/
    ├── Resume - Candidate A.markdown
    ├── Resume - Candidate B.markdown
    └── ...
```

### 4. Run the notebook

Open `Suffian_CO_Autonomous_HR_Agent.ipynb` and run cells top to bottom. The pipeline is organized in parts:

- **Part 1** — Load data, initialize LLM, run `ResumeIntelligenceAgent` over all resumes → `processed_resumes.json`
- **Part 2 & 3** — Load the job description, initialize `DecisionEngineAgent`, score every candidate, generate the ranked report and performance metrics → `candidate_decisions.json`, `decision_summary.json`

Each part can be re-run independently — later cells reload exported JSON if earlier variables aren't in memory, so you don't need to re-run everything from scratch every session.

## ⚙️ Configuration

A few knobs you'll likely want to tune before running against your own data:

| Setting | Where | Default | What it does |
|---|---|---|---|
| `use_groq` | `setup_llm(use_groq=...)` | `False` | Switch between local Ollama and hosted Groq |
| `model` (Ollama) | `setup_llm()` | `llama3.2` | Any locally-pulled Ollama model works |
| `model` (Groq) | `setup_llm()` | `llama-3.3-70b-versatile` | Any Groq-hosted model |
| `advance_threshold` | `DecisionEngineAgent.__init__` | project default | Minimum score to auto-advance a candidate |
| `maybe_threshold` | `DecisionEngineAgent.__init__` | project default | Minimum score to trigger phone screening instead of rejection |
| `search_kwargs={"k": ...}` | retriever setup (if using RAG search) | `4` | Number of chunks retrieved per query |

Adjust the thresholds based on how conservative or aggressive you want the auto-advance behavior to be — a smaller gap between `advance_threshold` and `maybe_threshold` means more candidates get flagged for phone screening rather than an automatic yes/no.

## 📊 Sample output

```
🥇 1. Jane Doe - 8.7/10
   🚀 Status: ADVANCE (High Priority)
   Action: Schedule technical interview
   Key Strengths: Strong Python background, relevant ML experience

🤔 2. John Smith - 6.2/10
   Status: MAYBE (Medium Priority)
   Action: Phone screening required
```

## 📈 Performance metrics (the agent tracks on itself)

Every run computes and prints a self-assessment, so you can see the efficiency case at a glance rather than taking it on faith:

- **Time efficiency** — automated seconds/candidate vs. a 15-min/candidate manual baseline, with total time saved
- **Cost savings** — time saved × configurable HR hourly rate, both in aggregate and per-candidate
- **Quality metrics** — % of candidates that received a valid (non-fallback) score, and score distribution (mean ± std dev)
- **System reliability** — % of candidates processed with full reasoning vs. those that hit the fallback path

This is the same data exported to `decision_summary.json`, so you can chart it or feed it into a dashboard separately.

## 🧭 Project status & roadmap

This repo currently implements the **screening and decision** stages of the pipeline (Parts 1–3). Planned next step:

- [ ] **Part 4 — Communication generation**: autonomously draft personalized interview invites / phone-screen requests / rejection emails per candidate, using the `data/templates/` folder and each candidate's decision record

Contributions and ideas for extending the pipeline (e.g. interview scheduling, ATS integration, bias auditing on scoring) are welcome.

## ⚠️ Notes & limitations

- Scoring quality depends heavily on the underlying LLM — local `llama3.2` via Ollama is free but less consistent than larger hosted models like Groq's `llama-3.3-70b`.
- The fallback scorer is a safety net, not a substitute for a good LLM response — treat MAYBE/fallback-scored candidates as needing human review, not as a firm signal.
- Resume and job-description parsing assumes reasonably clean Markdown input; scanned/PDF resumes would need an OCR or conversion step first.
- This is a learning/portfolio project, not a production HR tool. Any real hiring pipeline using LLM scoring should include human review, bias/fairness auditing, and legal review before making decisions that affect real candidates — automated scoring of protected characteristics or proxies for them can create real liability.

## ❓ FAQ

**Do I need an OpenAI or Groq API key to run this?**
No — by default it runs fully locally and for free via Ollama. Groq is optional if you want faster or larger-model inference.

**What format do resumes need to be in?**
Markdown (`.markdown` / `.md`) plain text. If you have PDFs or DOCX resumes, convert them to Markdown/plain text first.

**Can I use a different LLM provider (OpenAI, Anthropic, etc.)?**
Yes — since the pipeline is built on LangChain's standard `.invoke()` interface, swapping in `ChatOpenAI`, `ChatAnthropic`, or any other LangChain chat model is a small change in `setup_llm()`.

**Why does a candidate sometimes get a generic "fallback" reasoning?**
That means the LLM's response couldn't be parsed as valid JSON, so the system fell back to keyword-based sentiment scoring rather than crashing. It's a signal that candidate should get a manual look.

**Is this connected to an ATS (Applicant Tracking System)?**
Not currently — resumes and job descriptions are loaded from local Markdown files. Integrating with an ATS API is a natural extension.

## 🤝 Contributing

This started as a personal project to explore autonomous agent design, but contributions are welcome:

1. Fork the repo and create a feature branch
2. Keep new agents consistent with the existing pattern (structured prompt → JSON parsing → graceful fallback → stats tracking)
3. Open a PR with a short description of what changed and why

Ideas especially welcome on: Part 4 (communication generation), bias/fairness auditing for the scoring step, and ATS integrations.

## 📄 License

MIT — feel free to fork and adapt.
