# Multi-Agent Code Generator

A three-agent **LangGraph** pipeline that takes a one-line project idea and
turns it into a working codebase on disk — no scaffolding template, no
boilerplate generator, just a planner, an architect, and a coder agent that
read and write real files.

![python](https://img.shields.io/badge/python-LangGraph-1C3C3C) ![llm](https://img.shields.io/badge/LLM-Groq%20(gpt--oss--120b)-F55036) ![agents](https://img.shields.io/badge/agents-planner%20%E2%86%92%20architect%20%E2%86%92%20coder-6f42c1)

---

## How it works

```
user_prompt
    │
    ▼
[planner]    → structured Plan (name, description, tech stack, features, file list)
    │
    ▼
[architect]  → TaskPlan: one or more concrete ImplementationTasks per file,
    │           each naming exact functions/variables/imports and how it
    │           connects to earlier tasks
    ▼
[coder]      → loops over every task with a LangGraph ReAct tool-using agent
    │           that reads the current file, then writes the full updated
    │           content — one task at a time until the plan is exhausted
    ▼
   END
```

- **Planner** (`planner_agent`) — calls the LLM with structured output
  (`Plan` schema) to turn the free-text prompt into a project spec.
- **Architect** (`architect_agent`) — expands that Plan into a `TaskPlan`:
  an ordered list of file-level implementation steps, each carrying forward
  context from prior steps so later files integrate correctly.
- **Coder** (`coder_agent`) — for each step, spins up a `create_react_agent`
  with file tools, reads the existing file (if any) for context, and writes
  the complete new file content. Loops via a conditional edge back to itself
  until `current_step_idx` reaches the end of the plan.

All state is validated with Pydantic (`agents/states.py`): `Plan`, `File`,
`ImplementationTask`, `TaskPlan`, `CoderState`.

---

## Generated files stay sandboxed

Every file operation goes through `safe_path_for_project()` in
`agents/tools.py`, which resolves the target path and rejects anything that
would escape `./generated_project/` — so the coder agent can't write outside
its working directory even if a task description tries to point it there.

Available tools: `write_file`, `read_file`, `list_files`,
`get_current_directory`, `run_cmd` (sandboxed shell command runner).

---

## Repository layout

```
├── agents/
│   ├── graph.py      # LangGraph wiring: planner → architect → coder loop
│   ├── states.py      # Pydantic schemas (Plan, File, TaskPlan, CoderState)
│   ├── prompts.py       # Prompt templates for planner/architect/coder
│   └── tools.py           # Sandboxed file tools used by the coder agent
├── main.py                  # CLI entry point
└── requirements.txt
```

Generated projects land in `./generated_project/`, created on demand.

---

## Setup

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# .env
echo "GROQ_API_KEY=your_key_here" > .env
```

## Run

```bash
python main.py
# Enter your project prompt: Build a colourful modern todo app in html css and js
```

Or run the graph directly for a fixed example prompt:

```bash
python agents/graph.py
```

**Options** (`main.py`):

| Flag | Default | Purpose |
| --- | --- | --- |
| `--recursion-limit`, `-r` | `100` | LangGraph recursion limit (raise for larger projects with many implementation steps) |

---

## Tech stack

- **LangGraph** — state machine orchestrating planner → architect → coder
- **LangChain** (`create_react_agent`) — the coder's tool-calling ReAct loop
- **Groq** (`langchain-groq`, model `openai/gpt-oss-120b`) — LLM backend for all three agents
- **Pydantic** — structured output schemas enforced on every LLM call

---

## Notes

- `set_debug(True)` / `set_verbose(True)` are enabled in `agents/graph.py`,
  so runs print full LangChain trace output to the console — useful for
  development, but worth turning off for a quieter CLI experience.
- No test suite, license file, or CI config is present yet.
