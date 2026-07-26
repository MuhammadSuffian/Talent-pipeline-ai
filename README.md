# Autonomous HR Agent 🤖📋

An autonomous, multi-agent recruitment system built with **LangChain** that reads resumes, scores candidates against a job description, and makes independent ADVANCE / MAYBE / REJECT hiring decisions — with reasoning, no human in the loop.

> Built as a hands-on exploration of autonomous agent design: chaining together perception (resume parsing), judgment (LLM-based scoring), and decision logic (thresholded hiring calls) into a single pipeline.

---

## 💡 Why this exists

Screening resumes is repetitive and time-consuming. HR teams routinely spend significant hours per hire on manual tasks — reading resumes, comparing them to a job description, and deciding who moves forward.

This project explores how far you can push **autonomy** in that pipeline: given a job description and a folder of resumes, can a set of LLM-driven agents independently extract candidate profiles, score them consistently, and generate a ranked shortlist — end to end, without a human reviewing each step?

## 🧠 How it works

The system is organized into two cooperating agents:

### 1. `ResumeIntelligenceAgent`
- Takes raw resume text (Markdown) as input
- Uses a structured LLM prompt to extract: name, years of experience, current title, key skills, education, and a short professional summary
- Returns clean, structured JSON per candidate — turning unstructured resumes into comparable data

### 2. `DecisionEngineAgent`
- Takes the structured candidate profiles + the job description
- Scores each candidate on 4 dimensions: **technical skills, experience, education, overall fit** (0–10 scale)
- Applies configurable thresholds to autonomously classify each candidate as:
  - 🚀 **ADVANCE** — schedule a technical interview
  - 🤔 **MAYBE** — phone screening required
  - ❌ **REJECT** — send rejection email
- Produces reasoning, strengths, concerns, and suggested interview focus areas for every decision
- Includes a **fallback scoring mechanism** (keyword/sentiment-based) if the LLM response can't be parsed as JSON — so the pipeline degrades gracefully instead of crashing
- Tracks its own performance: processing time, decision distribution, and estimated time/cost savings vs. manual screening

Both agents log their own stats, and results are exported to JSON at each stage (`processed_resumes.json`, `candidate_decisions.json`, `decision_summary.json`) so later stages can pick up where earlier ones left off — even across separate notebook runs.

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

Plus autonomous performance metrics: total processing time, estimated time saved vs. manual screening, and cost-savings estimates.

## 🧭 Project status & roadmap

This repo currently implements the **screening and decision** stages of the pipeline (Parts 1–3). Planned next step:

- [ ] **Part 4 — Communication generation**: autonomously draft personalized interview invites / phone-screen requests / rejection emails per candidate, using the `data/templates/` folder and each candidate's decision record

Contributions and ideas for extending the pipeline (e.g. interview scheduling, ATS integration, bias auditing on scoring) are welcome.

## ⚠️ Notes & limitations

- Scoring quality depends heavily on the underlying LLM — local `llama3.2` via Ollama is free but less consistent than larger hosted models.
- The fallback scorer is a safety net, not a substitute for a good LLM response — treat MAYBE/fallback-scored candidates as needing human review.
- This is a learning/portfolio project, not a production HR tool. Any real hiring pipeline using LLM scoring should include human review and bias/fairness auditing before making decisions that affect real candidates.

## 📄 License

MIT — feel free to fork and adapt.
