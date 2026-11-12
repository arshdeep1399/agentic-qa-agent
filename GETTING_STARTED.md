# Getting started

About 20 minutes from a fresh machine to your first report. Commands are shown for **macOS/Linux** and
**Windows (PowerShell)**.

## 1. Install the tools

- **Python 3.11+** from python.org. On Windows, tick **"Add python.exe to PATH"** in the installer.
- **VS Code** (optional) with the Microsoft **Python** extension.

## 2. Set up the agent

From this folder:

```bash
python3 -m venv .venv
source .venv/bin/activate                 # Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
python -m playwright install chromium
python -m pytest -q                       # expect: 10 passed
```

On Windows, if activation says scripts are disabled, run
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once.

## 3. Add an API key (optional)

```bash
cp .env.example .env                      # Windows: copy .env.example .env
```

Put **one** key in `.env`:
- `GROQ_API_KEY=...`: free tier at console.groq.com, good for prototyping
- `ANTHROPIC_API_KEY=...`: Claude

Check it with `python -m qa_agent --check-llm`. The agent picks a model your key can use automatically.
`.env` is git-ignored, so never commit it or show it in a screenshot.

No key? Add `--offline` to every run.

## 4. Run the target app

This repository does **not** include the app under test. The agent was built for **DemoBank**, the
target app from the CDF Agentic QA Hackathon kit. Run your copy of the kit by following its
`LOCAL_SETUP.md` (PostgreSQL via Docker, a Python backend, a Vite frontend). When it's up:

- the website is at `http://localhost:5173` (use `localhost`, not `127.0.0.1`)
- the spec is the kit's `test-website/PRD.md`

## 5. Run the agent

```bash
python -m qa_agent --url http://localhost:5173 --prd path/to/test-website/PRD.md --headed
```

The first lines should read:
```
[interpret] 36 requirements extracted from PRD.md ...
[plan] 42 scenarios: positive=17, negative=12, edge=13
[preflight] target reachable; API discovered at http://localhost:8000 (verified)
```
If preflight prints a WARNING instead, add `--api-url http://localhost:8000`.

A full run takes about 3 minutes. The report is written to `runs/run-<timestamp>/report.html`.

**In VS Code:** open the Run and Debug panel, choose **QA agent: full run (watch browser)** and press F5.
It asks for the app URL and the spec path.

## Options

| Flag | What it does |
|---|---|
| `--headed` | Show the browser window |
| `--only transfers HIS-` | Run only some suites or scenario-ID prefixes |
| `--chaos` | Rename form labels at runtime (resilience demo) |
| `--offline` | Never call an LLM |
| `--provider groq\|anthropic` | Force a provider (default: whichever key is set) |
| `--api-url URL` | Set the backend URL if it isn't discovered automatically |
| `--out DIR` | Where to write runs (default `runs/`) |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `No module named qa_agent` | Run from this folder |
| `No module named playwright` | Activate `.venv` first |
| `Executable doesn't exist … chromium` | `python -m playwright install chromium` |
| `[preflight] target not usable` | Open the URL in a normal browser; check the app is running |
| Many **unverified** results | Add `--api-url <backend URL>` |
| `API error 401` in the report's fallbacks | Check the key in `.env` (no spaces or quotes) |
| `API error 429` | Provider rate limit: wait a minute and rerun (answers are cached) |
