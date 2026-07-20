# Data Validation Agent

## Architecture

```
Natural language request ("validate src vs tgt")
            |
            v
    Agent orchestrator (Groq LLM, tool-use loop)
       /         |          \
      v           v            v
 Inspect schema  Sample rows  Classify columns
 get_table_      get_sample_  NK vs audit vs
 columns         rows         business_data
       \          |          /
        v         v         v
        Postgres (read-only) — SELECT only, schema + samples
                  |
                  v
          Human confirmation
        Approve or edit NK/audit split
                  |
                  v
          SQL template renderer
          Deterministic, no LLM
                  |
                  v
          .sql file + summary
        Not executed, plain-language findings
```

## What this version changes
- **Runs on Groq.** The orchestrator (`ai_agent.py`)
  and the classifier (`classifier.py`) both call Groq's OpenAI-compatible
  chat completions API. Tool schemas in `tools.py` use the OpenAI/Groq
  function-calling format. Default model is `openai/gpt-oss-120b`,
  overridable via `GROQ_MODEL`.
- **Credentials live in a `.env` file, not shell exports.** `config.py`
  loads `agent/.env` automatically via `python-dotenv` at import time, so
  you set values once instead of re-exporting them in every new terminal
  session. See Setup below.
- **No arbitrary query execution.** There is no "run any SELECT" tool.
  The agent can only look up schema metadata and pull a small, bounded
  sample of rows (capped at 20). It never runs the validation query itself.
- **Human confirmation is a real, enforced gate — not just a prompt
  instruction.** After proposing a natural-key/audit classification, the
  model literally cannot call the render tool until a genuine new message
  from the user has arrived. This is enforced in `tools.py` via
  `AgentState`, independent of whether the model "remembers" to ask
  first. See "How the confirmation gate works" below.
- **SQL generation is deterministic.** `sql_renderer.py` builds the row
  count check and data validation check from plain Python string
  templates — no LLM is involved in producing the actual SQL text. The
  model's job is classification and conversation, not writing the final
  query.
- **The classifier is its own isolated component.** `classifier.py` is a
  separate, focused Groq call (not part of the orchestrator's main
  conversation) that only classifies columns given metadata and sample
  rows, using JSON response mode for reliable parsing.
- **The agent never reports results — because it never has any.** Since
  it doesn't execute the generated SQL, its final summary describes what
  the scripts check and where they were saved, not what they found. You
  run the `.sql` files yourself to see actual results.

## File layout
```
agent/
  .env               your real credentials (gitignored, never committed)
  config.py           connection/model settings, loaded from .env
  db_tools.py          read-only DB connection: schema lookup + bounded sample rows only
  classifier.py         isolated Groq call that proposes NK/audit/business_data splits
  sql_renderer.py        deterministic SQL template builder — no LLM
  sql_writer.py          writes rendered SQL to disk
  tools.py              tool schemas + dispatcher + AgentState (the confirmation gate)
  ai_agent.py            orchestrator: conversation history + tool-use loop (Groq)
  main.py               interactive CLI entry point
  webapp.py              hosted-style web chat interface (login, rate limiting, per-session state)
  templates/             chat.html, login.html
  requirements.txt
```

## How the confirmation gate works

`AgentState` (in `tools.py`) tracks two things per conversation:
- `awaiting_confirmation` — set to `True` when `classify_columns` returns
  a proposal
- `confirmed_this_turn` — set at the *start* of every new user message via
  `state.start_new_turn()`, and only ever `True` if a proposal was pending
  when that new message arrived

`render_sql_template` checks `confirmed_this_turn` before doing anything.
Since that flag is only set when a genuinely new user message begins, the
model cannot call `classify_columns` and `render_sql_template` back to
back in the same turn — the tool call itself is rejected with an error,
regardless of what the model intends. After a successful render, the gate
resets, so a second render needs a fresh classification and confirmation.

This was tested directly (bypassing the live API) to confirm all four
cases behave correctly: blocked before any proposal, blocked in the same
turn as a proposal, allowed after a genuine new turn, and blocked again
immediately after a successful render.

## Setup

1. Create the read-only role (run once, as your master user):
```
   psql ... -f 05_readonly_role.sql
```
   (or run it in DBeaver — same file, one level up from `agent/`)

2. Install dependencies:
```bash
   cd agent
   pip install -r requirements.txt
```

3. Create your `.env` file:

   Then edit `agent/.env`:
```
   PGHOST=source-and-target-db.cpckkgcwoy32.ap-south-1.rds.amazonaws.com
   PGPORT=5432
   PGDATABASE=source_target_db
   PGUSER=validation_agent
   PGPASSWORD=the-password-you-set-in-05_readonly_role.sql
   GROQ_API_KEY=gsk_...

   # only needed for webapp.py
   AGENT_PASSWORD=pick-a-password-for-the-web-login
   SECRET_KEY=paste-output-of-python-secrets-token-hex-32
```
   Generate a `SECRET_KEY` with:
```bash
   python -c "import secrets; print(secrets.token_hex(32))"
```
   `.env` is already listed in `.gitignore` — it will never be committed.

4. Run it:
```bash
   python main.py
```
   or as a local web chat:
```bash
   python webapp.py
```
   then open `http://127.0.0.1:5000`, log in with `AGENT_PASSWORD`, and
   use the same propose-then-confirm flow in the browser.

## Example session

```
You: Compare target_schema.stock_yearly_summary_expected against
     target_schema.stock_yearly_summary.

Agent: I looked at both tables — they have no primary key defined, so
I inferred the natural key from the column names. Here's what I'm
proposing:

  Natural key: symbol, trade_year
  Audit fields: (none found)
  Business data to compare: company_name, industry, trading_days,
  avg_open, avg_close, year_high, year_low, total_volume

Does this look right, or would you like to adjust it?

You: That looks right, go ahead.

Agent: Done. I've generated two scripts:
  - row_count_check.sql — compares total row counts between the two tables
  - data_validation_check.sql — compares the 8 business columns listed
    above, excluding symbol and trade_year since those identify the
    record rather than being data to validate

Both are saved in validation_output/. I haven't run them — open them in
DBeaver (or your SQL client of choice) to see the actual results.
```

## Known limitations
- Single shared login password for `webapp.py`, not a multi-user system.
- In-memory rate limiting and session state — both reset if the process
  restarts.
- The agent never executes the SQL it generates; you always run the
  `.sql` files yourself against the database to see actual results.

## Note on writing to mismatch_report
This agent only ever reads schema and samples — it has no path to
`INSERT` anything, including into `target_schema.mismatch_report`. That
logging step (see `03_validation_checks.sql`) stays a separate, manual
process: run the generated `.sql` files yourself, and if you want the
findings logged, insert them using your existing write-capable
credentials, kept deliberately separate from this agent's read-only role.
