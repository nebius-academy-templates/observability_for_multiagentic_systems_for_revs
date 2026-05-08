In this task, you'll connect the LEVEL_4 code review agent to **LangSmith** and use it to observe every node execution, LLM call, and conditional routing decision in real time.

You'll create a LangSmith project, configure the credentials, add run tags to distinguish scenarios, and then trace three distinct runs to understand what observability looks like in a LangGraph pipeline.

## Workflow at a glance
👩‍💻

1. Create a LangSmith project
2. Configure credentials
3. Add run tags to [main.py](http://main.py/)
4. Run the three scenarios
5. Explore the traces
6. Submit your task

### Button text: Let's go!

## 1. Create a LangSmith project

1. Go to [smith.langchain.com](https://smith.langchain.com/) and sign in (or create a free account).
2. Click **New Project** and name it `level-4-code-review`.
3. Open the project and confirm it is empty — no runs yet.
4. Go to **Settings → API Keys** and create a new API key. Copy it — you will not be able to see it again.

## 2. Configure credentials

Open the `.env` file in the project root and set the following four variables:

```
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=lsv2_your_key_here
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
LANGCHAIN_PROJECT=level-4-code-review
```

Replace `lsv2_your_key_here` with the key you copied in Step 1.

`LANGCHAIN_TRACING_V2=true` is the single flag that activates automatic tracing for all LangChain and LangGraph runs. Once it is set, every node execution, LLM call, and conditional edge decision is captured without any code changes to your graph.

Verify that `python-dotenv` loads the file by running:

```
python -c "import os; from dotenv import load_dotenv; load_dotenv(); print(os.getenv('LANGCHAIN_PROJECT'))"
```

Expected output: `level-4-code-review`

## 3. Add run tags to [main.py](http://main.py/)

Right now `main.py` passes a `metadata` dict to the graph. Extend the config to also include **tags**, so each scenario is easy to filter in LangSmith.

Find the `config` dict inside the `run()` function and add a `tags` key:

```python
config = {
    "tags": [label],
    "metadata": {
        "filename": label,
        "run_id": run_id,
    },
}
```

The `label` argument is already `"buggy diff"` or `"clean diff"` — it will appear as a tag on every run in the LangSmith UI, making it trivial to filter one scenario from the other.

## 4. Run the three scenarios

You will run `main.py` three times to produce three distinct traces.

### Scenario A — security blocked (buggy diff)

Run normally:

```
python main.py
```

This produces two traces in one run. Focus on the **buggy diff** trace. It should be blocked by `security_gate` before reaching `summarizer`.

### Scenario B — full pipeline (clean diff)

The second trace produced by the same run covers `clean.diff`. It should pass through `security_gate` and reach `summarizer`.

### Scenario C — force a retry loop

To trigger the `quality_router` retry branch, temporarily lower the score threshold in `graph.py`:

```python
# change this line inside quality_router:
if score >= 8:
```

to:

```python
if score >= 10:
```

Run again:

```
python main.py
```

This forces the clean diff to loop back to `bug_detector` until `retry_count` reaches 2. **Restore the threshold to `>= 8` after the run.**

## 5. Explore the traces

Open [smith.langchain.com](https://smith.langchain.com/), navigate to `level-4-code-review`, and confirm that traces are appearing.

### What to verify for Scenario A (buggy diff — blocked)

- The run is tagged `buggy diff`
- The node sequence ends at `security_auditor` — `summarizer` is absent
- Click into `security_auditor` → expand the **Output** panel → confirm `security_issues` is a non-empty list
- Click into any LLM call inside `security_auditor` → inspect the **System prompt** and the raw model response

### What to verify for Scenario B (clean diff — full pipeline)

- The run is tagged `clean diff`
- The node sequence includes `summarizer` after `security_auditor`
- Click into `security_auditor` → confirm `security_issues` is `[]`
- Click into `summarizer` → confirm `score` and `summary` are present in the output
- Click into `quality_router` (shown as a routing step) → confirm it returned `"END"`

### What to verify for Scenario C (retry loop)

- The same nodes appear **twice** in the trace: `bug_detector`, `style_reviewer`, `improvement_advisor`, `security_auditor`, `summarizer`
- The `retry_count` value in `summarizer`'s output increases from `1` to `2` across the two passes
- The run ends after the second pass because `retry_count >= 2` forces the exit

### LangSmith panels reference

| Panel | What you find there |
| --- | --- |
| **Timeline** | Visual sequence of all node and LLM calls with duration |
| **Input / Output** | Full state snapshot entering and leaving each node |
| **Metadata** | `filename` and `run_id` values passed from `main.py` |
| **Tags** | `buggy diff` or `clean diff` label for filtering |
| **LLM calls** | Raw system prompt, user message, and model response for every `_call()` invocation |

### Button text: Ready to submit

## 6. Submit your task

Before submitting, review the checklist.

### ✅ Submission checklist

- [ ]  LangSmith project `level-4-code-review` exists and is accessible
- [ ]  `main.py` — `"tags": [label]` added to the `config` dict
- [ ]  Scenario A trace: run tagged `buggy diff`, `summarizer` absent, `security_issues` non-empty
- [ ]  Scenario B trace: run tagged `clean diff`, `summarizer` present, `security_issues` is `[]`
- [ ]  Scenario C trace: node sequence shows two full passes, `retry_count` reaches `2`
- [ ]  `quality_router` threshold restored to `>= 8` after Scenario C
- [ ]  Changes are committed and pushed to `main`

1. Commit your changes.
2. Push to GitHub.
3. Return to the lesson and click "Submit."