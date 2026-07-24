# CareerPilot AI

CareerPilot is a small pipeline of Jupyter notebooks that takes a resume, figures out what job postings actually match it, ranks them, and explains *why* each one is a good fit — plus a quick breakdown of which skills you're missing for any job you paste in.

I built this to move past the usual "keyword-match resume screener" idea. Instead of just checking if a resume contains the word "Python," it parses the resume into a real profile, pulls live job postings for that specific person's skills (not a generic list), ranks them using sentence embeddings, and then uses a local LLM to write a short human-readable explanation for each match.

Everything runs as a sequence of five notebooks. There's no web app or frontend — each notebook is a stage, and they hand off data to each other through a SQLite database and a couple of JSON files.

## How it fits together

```
resume (PDF/DOCX)
   │
   ▼
1_resume_parsing.ipynb        →  parses text, extracts name/skills/contact info → careerpilot.db
   │
   ▼
2_job_fetching.ipynb          →  pulls live postings from Adzuna, personalized to the profile → job_dataset.json
   │
   ▼
3_job_matching.ipynb          →  embeds profile + jobs, ranks by cosine similarity → top_matches.json
   │
   ▼
4_generate_explanations_FIXED.ipynb  →  local Ollama model writes "why this fits" → final_recommendations.json
   │
   ▼
5_skill_gap_analysis.ipynb    →  compares profile skills against any job description, shows what's missing
```

Notebooks 1–4 are meant to run in order the first time. After that, 3, 4, and 5 can be re-run independently as long as the files upstream of them already exist. Notebook 5 doesn't even need Ollama — it's a plain set comparison, so it still works if the LLM step is slow or down.

## What each notebook actually does

**1. Resume Parsing** — Upload a PDF or DOCX through an `ipywidgets` file picker (there's a manual fallback if the widget doesn't render in your notebook environment). Text gets pulled out with `pdfplumber` / `python-docx`, then spaCy + a phrase matcher pull out name, organizations, and skills. Contact info (email/phone) is grabbed with regex. Everything gets written as a row in `careerpilot.db`, along with a `profile_completion` percentage based on how many fields were actually filled in.

**2. Job Fetching** — Hits the Adzuna Jobs API and saves results to `job_dataset.json`. This used to just search 4 hardcoded terms so every resume got the same 34 jobs back — that's fixed now, it reads the latest profile from `careerpilot.db` and builds search terms out of that person's actual skills and career goal. If no profile exists yet it falls back to generic terms, so the notebook still runs on its own.

**3. Job Matching** — Loads the profile and the job postings, embeds both with `sentence-transformers`, and ranks jobs by cosine similarity. This is retrieval only, nothing gets trained. It also does a sanity check that the job pool it's ranking against was actually fetched for the current profile (not a leftover from someone else's run) and warns if not.

**4. Generate Explanations** — Takes the top matches and asks a local Ollama model (`qwen3.5:4b` in the current version, swap for whatever you've pulled) to write a couple of sentences per job: why it's a good fit, and one skill to work on next. There's a small warm-up call first since the first request to a cold model on a shared box can take a while. `think` mode is explicitly turned off for this model — leaving it on made a normal explanation call take minutes instead of seconds.

**5. Skill Gap Analysis** — Paste any job description in, and it diffs the required skills against the candidate's parsed skill list — what's matched, what's missing. Works against a pasted JD or the #1 Adzuna match from step 3.

## Repo layout

```
Career Copilot/
  1_resume_parsing.ipynb
  2_job_fetching.ipynb
  3_job_matching.ipynb
  4_generate_explanations_FIXED.ipynb
  5_skill_gap_analysis.ipynb
  careerpilot.db              # SQLite DB, single table: career_profiles
  job_dataset.json            # output of step 2
  top_matches.json            # output of step 3
  final_recommendations.json  # output of step 4
  uploads/                    # sample resumes used to test the pipeline


```



## Setup

```bash
pip install pdfplumber python-docx spacy ipywidgets requests sentence-transformers numpy
python -m spacy download en_core_web_sm
```

(Each notebook also has its own `pip install` cell at the top for whatever it specifically needs, so you don't have to install everything up front if you're only running one stage.)

You'll also need:
- An **Adzuna API** app ID + key (free tier is fine) for notebook 2 — get one at [developer.adzuna.com](https://developer.adzuna.com/).
- **Ollama** running locally with a model already pulled, for notebook 4. Check what you have with `ollama list`. If you're on a shared/restricted box, you can't necessarily pull new models, so just pick whatever's already there and update `OLLAMA_MODEL`.



### Running it

Open the notebooks in Jupyter and run top to bottom, in order, starting with `1_resume_parsing.ipynb`. Each one prints what it saved and where, and tells you which notebook to run next if something's missing.

## Known rough edges

- The skill keyword list (`SKILLS = [...]`) is copy-pasted into three different notebooks (1, 2, and 5) instead of living in one shared module. They're kept in sync by hand right now — add a skill in one place, remember to add it everywhere. Worth pulling into a shared `skills.py` at some point.
- "Latest profile" is determined by `ORDER BY id DESC LIMIT 1` on the whole table — there's no concept of a logged-in user or session, so if two people parse resumes back to back, the second one's profile is what everything downstream uses. Fine for a demo, not fine for anything with more than one user.
- Education extraction in step 1 isn't implemented yet — the field exists in the DB schema but is saved as an empty list.
- No automated tests. Everything's been checked by running the notebooks end to end against a handful of real resumes in `uploads/`.

## Tech used

Python, spaCy, pdfplumber, python-docx, sentence-transformers (embeddings + cosine similarity for matching), SQLite, the Adzuna Jobs API, and Ollama for local LLM generation.
