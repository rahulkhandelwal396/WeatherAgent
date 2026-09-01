# Weather Agent

A minimal, local ReAct-style weather agent built on top of Ollama and the Open-Meteo API. The agent reasons about a user's question, decides to call a `get_weather` tool, executes that tool for real, and then produces a clean, natural-language final answer using only the returned data.

No cloud LLM API key is required. No weather API key is required. Everything runs locally against an Ollama model and a free, keyless weather API.

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Example Run](#example-run)
- [Design Notes](#design-notes)
- [Limitations](#limitations)
- [Possible Extensions](#possible-extensions)
- [License](#license)

## Overview

This project implements a small but complete ReAct (Reason and Act) agent loop from scratch, without an agent framework. It is meant as a learning exercise and reference implementation for how tool-calling agents actually work under the hood: prompting, stopping generation before the model can hallucinate a tool result, executing the real tool, feeding the result back into the conversation, and extracting a clean final answer.

The agent answers a single hardcoded question in its current form (`What's the weather in London?`), but the flow is written so any question can be substituted.

Core pieces:

- A local LLM served through Ollama (`qwen3:4b` by default).
- A single tool, `get_weather`, backed by the free Open-Meteo geocoding and forecast APIs.
- A two-call LLM flow: one call to decide the action, one call to produce the final answer.
- Strict parsing and validation at every step, with clear error messages when something goes wrong.

## How It Works

The agent follows the classic ReAct pattern: Thought, Action, Observation, Final Answer. Concretely, this notebook enforces it as an explicit two-turn conversation rather than trusting the model to self-regulate:

1. **Check Ollama.** Confirm the Ollama server is reachable and the configured model is installed.
2. **First LLM call.** Send the system prompt and the user question. Generation is stopped at the string `Observation:` so the model cannot invent a fake weather result.
3. **Parse the Action.** Scan the model's output for the first valid JSON object containing both an `action` and an `action_input` key.
4. **Validate the Action.** Confirm `action` is `get_weather` and `action_input.location` is a non-empty string.
5. **Execute the real tool.** Call `get_weather(location)`, which geocodes the location and fetches current conditions from Open-Meteo.
6. **Extend the conversation.** Append the model's own Thought/Action output as an assistant turn, then append the real Observation as a new user turn (the Observation is never merged into the assistant's own message).
7. **Second LLM call.** Ask the model to produce a `Final Answer:` using only facts from the Observation. This call has no stop sequence, so the model is free to finish its reasoning naturally.
8. **Extract the Final Answer.** Take everything after the *last* occurrence of `Final Answer:` in the raw output, since small local models sometimes mention that phrase while reasoning about the required format before actually producing it.

```
 User question
      |
      v
 [ LLM call #1 ]  --stop at "Observation:"-->  Thought + Action JSON
      |
      v
 Parse + validate Action JSON
      |
      v
 [ get_weather(location) ]  -->  Open-Meteo geocoding + forecast APIs
      |
      v
 Append assistant turn (Thought + Action)
 Append user turn (real Observation + instructions)
      |
      v
 [ LLM call #2 ]  -->  raw output containing "Final Answer: ..."
      |
      v
 Extract text after the last "Final Answer:"
      |
      v
 Final Answer printed to console
```

## Project Structure

```
.
├── Weather_Agent.ipynb   # Full implementation, runnable top to bottom
└── README.md
```

The notebook is organized into numbered sections that mirror the flow above:

| Section | Purpose |
|---|---|
| 1. Configuration | Ollama endpoints, model name, timeouts, token limits |
| 2. System Prompt | Defines the ReAct response format and tool contract |
| 3. Final Answer Instruction | Strict formatting instructions sent alongside the real Observation |
| 4. Output Formatting | `print_section` helper for readable console output |
| 5. Check Ollama | Verifies the server is running and the model is installed |
| 6. Real Weather Tool | `get_weather`, geocoding + current conditions via Open-Meteo |
| 7. Call Ollama | `call_llm`, wraps `POST /api/chat` with stop sequences and sampling options |
| 8. Extract Action JSON | `extract_action_json`, scans for the first valid `{action, action_input}` object |
| 9. Validate Action | `validate_action`, enforces the tool contract before execution |
| 10. Extract Final Answer | `extract_final_answer`, isolates the final user-facing sentence |
| 11. Main Agent Flow | `main()`, orchestrates the full two-call loop |
| 12. Run | Script entry point |

## Requirements

- Python 3.10 or later
- [Ollama](https://ollama.com/) installed and running locally
- The `qwen3:4b` model pulled in Ollama (or another model, see [Configuration](#configuration))
- Python packages:
  - `requests`
  - `huggingface_hub` (installed in the first notebook cell; not required by the core agent logic itself)

Internet access is required only to reach the Open-Meteo APIs, both of which are free and keyless.

## Installation

1. **Install Ollama** and start the server:

   ```bash
   ollama serve
   ```

2. **Pull the model** used by the agent:

   ```bash
   ollama pull qwen3:4b
   ```

3. **Clone this repository:**

   ```bash
   git clone https://github.com/<your-username>/weather-agent.git
   cd weather-agent
   ```

4. **Install Python dependencies:**

   ```bash
   pip install requests huggingface_hub
   ```

5. **Launch Jupyter and open the notebook:**

   ```bash
   jupyter notebook Weather_Agent.ipynb
   ```

## Usage

Run all cells in `Weather_Agent.ipynb` from top to bottom. The last cell calls `main()`, which:

1. Verifies Ollama is running and `qwen3:4b` is installed.
2. Asks the hardcoded question, `What's the weather in London?`
3. Prints each stage of the agent's reasoning as it happens: the raw first LLM output, the parsed Action JSON, the requested location, the real Observation from Open-Meteo, the full message history before the second LLM call, and the final clean answer.

To ask a different question, change the `user_question` variable inside `main()`:

```python
user_question = "What's the weather in Tokyo?"
```

The agent will resolve any location name that Open-Meteo's geocoding API can recognize, not just the cities used in testing.

## Configuration

All tunable values live at the top of the notebook, in the Configuration section:

| Variable | Default | Description |
|---|---|---|
| `OLLAMA_CHAT_URL` | `http://localhost:11434/api/chat` | Ollama chat completion endpoint |
| `OLLAMA_TAGS_URL` | `http://localhost:11434/api/tags` | Ollama endpoint used to list installed models |
| `MODEL` | `qwen3:4b` | Ollama model used for both LLM calls |
| `REQUEST_TIMEOUT_SECONDS` | `20` | Timeout for calls to the Open-Meteo APIs |
| `OLLAMA_TIMEOUT_SECONDS` | `180` | Timeout for calls to Ollama |
| `MAX_PREDICTED_TOKENS` | `300` | `num_predict` cap passed to Ollama for each call |

Both LLM calls use `temperature=0` and `think=False` for deterministic, non-thinking-mode output. Swap `MODEL` for any chat-capable model available in your local Ollama installation; no other code changes are required as long as the model can follow the ReAct-style formatting instructions in the system prompt.

## Example Run

Console output for the question `What's the weather in London?`:

```
======================================================================
INSTALLED OLLAMA MODELS
======================================================================
- qwen3:4b

======================================================================
FIRST LLM OUTPUT
======================================================================
Thought: I need current weather data for London.
Action:
{"action": "get_weather", "action_input": {"location": "London"}}

======================================================================
PARSED ACTION
======================================================================
{
  "action": "get_weather",
  "action_input": {
    "location": "London"
  }
}

======================================================================
LOCATION REQUESTED BY MODEL
======================================================================
London

======================================================================
REAL OBSERVATION FROM OPEN-METEO
======================================================================
{
  "location": "London",
  "country": "United Kingdom",
  "temperature_c": 27.1,
  "feels_like_c": 24.8,
  "humidity_percent": 28,
  "precipitation_mm": 0.0,
  "wind_speed_kmh": 10.8
}

======================================================================
FINAL ANSWER
======================================================================
It's 27.1°C in London, feeling like 24.8°C, with 28% humidity, no precipitation, and wind at 10.8 km/h.
```

Note that this example reflects a real run against live weather data, so actual temperatures and conditions will differ when you run the notebook yourself.

## Design Notes

A few implementation choices are worth calling out, since they are the parts most likely to trip up a first attempt at building a tool-calling agent from scratch:

- **Stop sequences prevent hallucinated Observations.** The first LLM call passes `stop=["Observation:"]` so the model cannot fabricate a weather result. The real Observation is always the one actually returned by `get_weather`.
- **The Observation is a separate user turn.** It is appended as its own message rather than merged into the assistant's Thought/Action message, preserving a clean `system, user, assistant, user, assistant` sequence that mirrors how most chat-based tool-calling APIs expect history to be structured.
- **The Action JSON is found by scanning, not by regex.** `extract_action_json` walks the text character by character looking for `{`, then attempts `json.JSONDecoder().raw_decode` at each position. This tolerates a model that emits reasoning text before or after the JSON object without needing a fragile regular expression.
- **The Final Answer is taken from the last match, not the first.** Small local models sometimes reference the phrase `Final Answer:` while reasoning about the output format before actually producing it. Using `rsplit` and taking everything after the final occurrence avoids picking up that false positive.
- **No weather API key.** Open-Meteo's geocoding and forecast endpoints are free and require no authentication, which keeps the whole project runnable with zero paid services.

## Limitations

- The agent supports exactly one tool (`get_weather`). It is not a general-purpose multi-tool agent.
- The user question is currently hardcoded in `main()`; there is no CLI or chat interface.
- There is no retry logic if the model fails to produce valid Action JSON or a Final Answer; a `ValueError` or `RuntimeError` is raised instead.
- Behavior depends on the chosen local model's ability to follow formatting instructions. Smaller or less instruction-tuned models may need prompt adjustments to reliably produce the expected `Thought` / `Action` / `Final Answer` structure.
- The notebook was developed and tested on Windows with a local Ollama installation; paths and shell commands in the setup cell reflect that environment.

## Possible Extensions

- Accept the user's question as a CLI argument or through a simple chat loop instead of hardcoding it.
- Add additional tools (for example, a forecast lookup or a units-conversion tool) and extend the system prompt's tool contract accordingly.
- Add retry or repair logic that re-prompts the model when Action JSON parsing or Final Answer extraction fails.
- Stream tokens from Ollama instead of using `stream=False` for a more interactive experience.
- Wrap the flow in a small function or class that returns the final answer instead of only printing it, so it can be used as a library rather than a script.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details, or add one if it is not yet present in this repository.
