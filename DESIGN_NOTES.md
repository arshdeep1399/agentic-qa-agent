# Design notes

Why the agent is built the way it is. Each section points to the file where the idea lives.

---

## 1. A click that works is not a pass

Most automated UI tests check "did the button click without an error?". DemoBank is built to punish
that. A negative-amount transfer shows a green **"Transfer complete"**, and that green message *is* the bug.

The real question is: **what is true in the system after the action, and does it match the spec?**
In testing this is the **oracle problem**: you need a source of truth to compare against.

The oracle has two parts (`flows.py`: `api_state`, `read_dashboard_cards`):
- what the **UI** shows (parsed from visible text)
- what the app's own **API** returns for the same browser session (discovered by watching its traffic)

Pattern used throughout `suites/transfers.py`:
```
before = snapshot()  →  act through the UI  →  after = snapshot()
assert the UI said the right thing   AND   the right things changed (and nothing else)
```

---

## 2. Requirements as data

`prd.py` turns prose into structured requirements with **stable IDs** (`TRF-04`), so results can be
compared across runs and every assertion records which requirement it covers.

Scenarios don't hard-code IDs. They declare an **anchor phrase** ("start and end dates are inclusive")
that is resolved by substring search. Reorder the spec and the mapping still holds; delete the sentence
and the planner warns.

**Roll-up rule** (`models.py`: `roll_up`):
- any failed assertion → **failed**
- all assertions passed → **passed**
- nothing reliable observed → **unverified** (never a silent pass)

---

## 3. Test-design techniques

| Technique | What it means | Where it's used |
|---|---|---|
| Equivalence partitioning | One test per group of inputs that should behave the same | amounts: blank, zero, negative, non-numeric, too precise |
| Boundary value analysis | Test exactly at and just past the edges | $0.01, exactly the balance (allowed), balance + $0.01 (must fail), start date = end date |
| Negative testing | Invalid input must be rejected *and leave no trace* | every rejection checks balances unchanged and no new record |
| State-transition testing | Behaviour depends on state | logout → back button → refresh → direct URL → replayed session cookie |
| Differential checking | Two views that must agree | dashboard vs detail vs API balance; filtered rows vs the filter applied to the full list |
| Fault injection | Simulate failures to test recovery | Playwright `route()` returns 500/503 inside the agent's own browser only |

Example: the **date-boundary** test picks a transaction stored at `2025-03-05 02:45 UTC`, which is
**4 March** in Pacific Time (the UI's stated zone). Testing start-only, end-only and start = end
separately showed that the *end* date is exclusive while the start date works.

---

## 4. Resilience

`browser.py`: `Driver.find(intent, *strategies)`
1. Try semantic strategies in order (label → placeholder → test-id …).
2. Remember which one worked and try it first next time.
3. If all fail, send the page's **ARIA snapshot** to a cheap model, which proposes `{role, name}`.
   The proposal is verified (it must be visible) before use.

`settle()` waits until the network is quiet, no loading indicator is visible, **and the screen has stopped
changing** (an identical fingerprint of the URL and main content across consecutive samples). Actions
that load data, like filtering, wait for that data request to return.

`--chaos` renames form labels at runtime. A full run gave the same verdicts on all 36 requirements, with
38 automatic recoveries.

**Retries need idempotency thinking.** A failed read-only scenario is retried in a fresh browser profile;
a scenario that already sent a transfer is **not**, because a retry could move money twice.

---

## 5. LLMs only where judgement is needed

| Job | Why an LLM | Cost control |
|---|---|---|
| Decompose the spec into atomic assertions and test ideas | Language understanding | Once per spec, cached on disk |
| "Is this error message a clear explanation?" | Subjective | One batched call for all messages |
| Heal a broken locator | Fuzzy matching on the ARIA tree | Only on failure |
| Executive summary and likely root causes | Writing | One call |

Everything else is deterministic, so verdicts are reproducible. With no API key, each job has a
fallback (heuristic or template), and the report lists which ones ran.

`llm.py` supports Groq and Claude, asks the provider which models the key can use and picks the best
available one, and logs every call's tokens and cost, priced at the provider's rate *and* as a
Claude-equivalent for comparison.

---

## 6. Bugs found in the agent itself

The agent is code, and code can be wrong. Each of these would have produced false results:

1. **The judge graded a success receipt.** When the app wrongly accepted a $0 transfer, the clarity check
   read the *success receipt* and passed it. Fix: only grade text from error elements, and only when the
   UI actually rejected the input.
2. **Stale data matched.** A check found a row left by a *previous* run. Fix: match on the unique memo
   `QA-<run-id>-<scenario>`.
3. **It worked in one environment and broke in another.** On a real dev setup (Vite dev server, React
   Router transitions), 21 requirements came back unverified: (a) API discovery mistook a *source file*
   (`/src/api/client.ts`) for the backend; (b) React briefly kept the old screen visible after navigation,
   so the agent read stale content; (c) the configured model wasn't available to the API key. Fixes: only
   count fetch/XHR JSON responses, wait for the screen to stop changing, and ask the provider for available
   models.
4. **The AI judge was too strict, and the summary overstated.** It marked "Enter an amount to transfer." as
   unclear for a blank amount, and the summary called correctly rejected inputs "accepted". Fixes: judge
   clarity *for the specific input*, and tag each failure (wrongly accepted / unclear message / wrong data)
   before asking for a summary.

Lesson: test the tester in the environment real users have, review every failure against its evidence,
and prefer an honest "unverified" over a guess.

---

## 7. Reports people can act on

Each defect has a severity, the requirement IDs, numbered reproduction steps, expected vs observed values
(e.g. `Everyday Checking $4369.75 → $-0.01`), screenshots and the raw API response.

Severity rule (`report.py`): **Critical** = money or records changed when they must not; **High** = a rule
violated without data corruption; **Medium** = wrong or missing display/filter data; **Low** = wording/UX.

---

## 8. Ideas to extend

- Run in CI on every deploy and diff reports between runs.
- Let the LLM propose *new* scenarios for requirements that are still unverified.
- Print a traceability matrix: requirement → covering scenarios.
- Add visual-regression checks alongside the behavioural ones.
