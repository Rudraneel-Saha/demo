# HealthBot: AI-Powered Patient Education System

A LangGraph-based prototype chatbot built for the MediTech Solutions capstone: patients pick a health
topic, get a Tavily-sourced, patient-friendly summary from reputable medical sites, take a one-question
comprehension check, and receive a cited grade — then can loop into a new topic or exit.

All of the logic lives in **`healthbot.ipynb`**.

## Setup

1. **Install [uv](https://docs.astral.sh/uv/)** if you don't have it, then from this project folder:

   ```bash
   uv venv --python 3.11.13
   ```

2. **Activate the virtual environment**

   - Windows (PowerShell): `.\.venv\Scripts\Activate`
   - macOS/Linux: `source .venv/bin/activate`

3. **Install dependencies** (already declared in `requirements.txt` / `pyproject.toml`):

   ```bash
   uv add -r requirements.txt
   ```

4. **Get API keys**
   - OpenAI: https://platform.openai.com/api-keys
   - Tavily (free tier, 1000 requests): sign up at https://app.tavily.com/home and grab your key from the
     same page.

5. **Create `config.env`** in this folder (copy `config.env.example`) and fill in your real keys:

   ```env
   OPENAI_API_KEY="sk-..."
   TAVILY_API_KEY="tvly-..."
   ```

   `config.env` is already listed in `.gitignore` — never commit real keys.

6. **Launch Jupyter and open `healthbot.ipynb`**:

   ```bash
   uv run jupyter notebook
   ```

   Run the cells top to bottom. The last cell starts an interactive session: it uses `input()` to prompt
   you (rendered as a modal text box in Jupyter) and `print()` to show results.

## How it works

The notebook builds a single LangGraph `StateGraph` over a `HealthBotState` TypedDict. Each node has one
responsibility:

| Node | Responsibility |
| --- | --- |
| `collect_topic` | Ask the patient for a topic; reset all per-topic state fields |
| `search_health_info` | The LLM calls the Tavily tool (via tool-calling), restricted to reputable medical domains (Mayo Clinic, CDC, NIH, MedlinePlus, WHO, etc.) |
| `summarize_results` | LLM writes a 3–4 paragraph patient-friendly summary, using only the Tavily results |
| `display_summary` | Print the summary |
| `await_ready_for_quiz` | Wait for the patient to signal they're ready |
| `generate_quiz_question` | LLM writes one quiz question, using only the summary |
| `display_quiz_question` | Print the question |
| `collect_quiz_answer` | Collect the patient's answer |
| `grade_quiz_answer` | LLM grades the answer (A–F) using only the summary, with a structured, cited justification |
| `display_grade` | Print the grade and feedback |
| `ask_continue_or_exit` | Ask whether to loop into a new topic or end the session |

The only cycle is `ask_continue_or_exit → collect_topic`, and `collect_topic` explicitly clears every
field from the previous topic (search results, summary, quiz, answer, grade) each time it runs, so a new
topic never carries over stale data from a previous one.

## Testing without live API calls

The workflow logic (state transitions, the topic-reset loop, tool-calling wiring) was validated with a
scripted test harness that substitutes fake LLM/Tavily responses — see the project's test notes if you'd
like to re-run that check. Running the notebook itself for real requires valid `OPENAI_API_KEY` and
`TAVILY_API_KEY` values.

## Rubric mapping

- **LangGraph Configuration** — keys loaded via `python-dotenv` with asserts; `search_health_info` binds
  the Tavily tool to the model and lets the model issue the tool call.
- **Model Summarization** — `summarize_results` prompts the model to use *only* the Tavily output, 3–4
  paragraphs, patient-friendly language.
- **Quiz Question Creation** — `generate_quiz_question` prompts the model to use *only* the summary.
- **Grading Response** — `grade_quiz_answer` uses `with_structured_output` to force a letter grade plus a
  justification that cites the summary.
- **State Management** — `HealthBotState` TypedDict threaded through every node; each node reads prior
  state and returns only its updates.
- **Nodes & Edges / Human Interaction** — single-responsibility nodes, sequential edges, a conditional
  loop-or-exit edge, and full patient interaction via `input()`/`print()`.
