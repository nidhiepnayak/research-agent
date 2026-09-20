# Research Agent

An autonomous research agent that takes a research topic, breaks it into sub-questions, searches for information, synthesizes findings, critiques its own output for research gaps, optionally performs another bounded research iteration, and compiles a cited Markdown report.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Agent Workflow](#agent-workflow)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Configuration](#configuration)
- [Demo Mode](#demo-mode)
- [Ollama Mode (Local LLM)](#ollama-mode-local-llm)
- [Web Search](#web-search)
- [Local Documents](#local-documents)
- [CLI Usage](#cli-usage)
- [Streamlit UI](#streamlit-ui)
- [Citations](#citations)
- [Testing](#testing)
- [Example](#example)
- [Limitations](#limitations)
- [Future Improvements](#future-improvements)
- [Portfolio Notes](#portfolio-notes)
- [Attribution](#attribution)
- [License](#license)

## Overview

This project explores agentic research automation: given a topic, the agent plans what it needs to find out, searches for information, synthesizes what it finds, critiques its own synthesis for gaps, and — within a bounded number of iterations — refines the research before producing a final cited report.

The orchestration is built as a state-based workflow using LangGraph, with a pluggable LLM layer that supports Anthropic, OpenAI, or a locally-run Ollama model.

## Architecture

```text
┌─────────────────────────────────────────────┐
│                 CLI / Streamlit UI            │
└───────────────────────┬───────────────────────┘
                         │
                 ┌───────▼────────┐
                 │     engine      │  orchestrates a run
                 └───────┬────────┘
                         │
                 ┌───────▼────────┐
                 │      graph      │  LangGraph state graph
                 └───────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                 │
 ┌──────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐
 │    nodes     │  │    state     │  │    prompts    │
 │ (workflow    │  │ (shared run   │  │ (LLM prompt   │
 │  steps)      │  │  data)        │  │  templates)   │
 └──────┬───────┘  └──────────────┘  └───────────────┘
        │
 ┌──────▼───────┐        ┌────────────────┐
 │    tools      │──────▶│  llm (provider  │
 │ web_search,   │        │  abstraction:   │
 │ fetch,        │        │  Anthropic/     │
 │ documents     │        │  OpenAI/Ollama) │
 └──────┬───────┘        └────────────────┘
        │
 ┌──────▼───────┐
 │    report     │  compiles final Markdown report
 └──────────────┘
```

| Component | Role |
|---|---|
| `config.py` | Loads configuration (LLM provider, model, API keys, run limits) |
| `state.py` | Defines the shared state object passed between graph nodes |
| `llm.py` | Abstracts calls to Anthropic, OpenAI, or Ollama behind a common interface |
| `prompts.py` | Prompt templates used at each stage of the workflow |
| `nodes.py` | Individual workflow steps (planning, searching, synthesizing, critiquing) |
| `graph.py` | Defines the LangGraph state graph connecting the nodes |
| `engine.py` | Runs a full research session through the graph |
| `report.py` | Compiles the final cited Markdown report |
| `utils.py` | Shared helper functions |
| `tools/web_search.py` | DuckDuckGo-based web search |
| `tools/fetch.py` | Fetches and parses web page content (requests / BeautifulSoup) |
| `tools/documents.py` | Loads local reference documents (PDF, Markdown, text) |

## Agent Workflow

```text
User Research Question
        │
        ▼
Load Local Documents
        │
        ▼
Plan Sub-Questions
        │
        ▼
Search Web  ◀────────────┐
        │                 │
        ▼                 │
Synthesize Findings        │
        │                 │
        ▼                 │
Critique Findings           │
        │                 │
        ▼                 │
   Gaps Found?             │
     │       │             │
    YES      NO            │
     │       │             │
     └───────┘             │
     (bounded by           │
   MAX_ITERATIONS) ─────────┘
        │
        ▼
Compile Markdown Report
```

The critique/refinement loop is **bounded** by `MAX_ITERATIONS` — it is not an unlimited autonomous loop. Once the limit is reached, or no further gaps are identified, the agent compiles the final report.

## Key Features

- Sub-question planning from a single research topic
- Web search per sub-question via a DuckDuckGo-based search tool
- Support for local reference documents (PDF, Markdown, text) alongside web search
- Synthesis of findings across multiple sources
- Self-critique step that identifies gaps in the current research
- Bounded refinement loop to address identified gaps
- Cited Markdown report generation
- Pluggable LLM provider: Anthropic, OpenAI, or local Ollama
- Demo mode for verifying the pipeline without API keys or network calls
- CLI and optional Streamlit UI
- Automated test suite

## Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | LangGraph |
| LLM Providers | Anthropic, OpenAI, Ollama (provider-abstracted) |
| Local LLM Execution | Ollama |
| Web Search | DuckDuckGo (via project's search tool) |
| Web Fetching / Parsing | requests, BeautifulSoup |
| Document Loading | PyPDF (PDF), Markdown, plain text |
| UI | Streamlit |
| CLI | Python (`cli.py`) |
| Testing | unittest / pytest |
| Language | Python |

## Project Structure

```text
research-agent/
├── app.py
├── cli.py
├── research_agent/
│   ├── __init__.py
│   ├── config.py
│   ├── engine.py
│   ├── graph.py
│   ├── llm.py
│   ├── nodes.py
│   ├── prompts.py
│   ├── report.py
│   ├── state.py
│   ├── utils.py
│   └── tools/
│       ├── __init__.py
│       ├── documents.py
│       ├── fetch.py
│       └── web_search.py
├── tests/
├── examples/
├── docs/
├── requirements.txt
├── requirements-dev.txt
├── requirements-ui.txt
├── LICENSE
└── README.md
```

## Installation

macOS / Linux:

```bash
git clone <repository-url>
cd research-agent
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Windows (PowerShell), activation step differs:

```powershell
git clone <repository-url>
cd research-agent
python3 -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

Optional dependencies:

```bash
pip install -r requirements-ui.txt   # Streamlit UI
pip install -r requirements-dev.txt  # testing / dev tools
```

## Configuration

The following environment variables are used by the project's configuration layer:

| Variable | Purpose |
|---|---|
| `LLM_PROVIDER` | Selects the LLM backend: `anthropic`, `openai`, or `ollama` |
| `LLM_MODEL` | Selects the specific model for the chosen provider |
| `ANTHROPIC_API_KEY` | API key, required when `LLM_PROVIDER=anthropic` |
| `OPENAI_API_KEY` | API key, required when `LLM_PROVIDER=openai` |
| `OLLAMA_HOST` | Host address for a running Ollama instance |
| `TAVILY_API_KEY` | API key for the Tavily search integration, if used |
| `MAX_RESULTS_PER_QUERY` | Maximum web search results retrieved per sub-question |
| `MAX_ITERATIONS` | Maximum number of critique/refinement iterations |
| `USE_LANGGRAPH` | Toggles use of the LangGraph-based orchestration path |

Do not commit API keys to version control. Set them as environment variables in your shell, or in a local `.env` file that is excluded via `.gitignore`.

## Demo Mode

Demo mode exercises the full pipeline using fake LLM and search responses. It requires no API keys and makes no network calls.

```bash
python cli.py "The impact of quantum computing on cryptography" --demo
```

Demo mode **does not perform real research** — it exists solely to verify that the orchestration pipeline runs correctly end to end.

## Ollama Mode (Local LLM)

Ollama lets the agent run against a local LLM instead of a paid external API.

1. Install Ollama from [ollama.com](https://ollama.com).
2. Pull a model:
```bash
   ollama pull llama3.2:3b
```
3. Configure the provider:
```bash
   export LLM_PROVIDER=ollama
   export LLM_MODEL=llama3.2:3b
```
4. Run the agent:
```bash
   python cli.py "The impact of quantum computing on cryptography" --max-results 2 --max-iterations 1 -v
```

`llama3.2:3b` is the model tested during development; other Ollama-compatible models can be used depending on available hardware.

## Web Search

The agent performs web search through its own search tool (DuckDuckGo-based) to gather information for each sub-question. This allows real research runs without requiring a paid search API when combined with a local Ollama model.

Search result quality and coverage can vary. Generated research should be checked against the underlying sources rather than treated as authoritative on its own.

## Local Documents

The agent can incorporate local reference material — PDF, Markdown, or text files — as additional context alongside web search.

Example:

```bash
python cli.py "Research question" --docs notes.pdf research.md
```

## CLI Usage

```bash
# Real run using a configured LLM provider
python cli.py "Your research topic" --max-results 2 --max-iterations 1 -v

# Demo run (no API keys or network calls)
python cli.py "Your research topic" --demo

# Include local documents alongside web search
python cli.py "Your research topic" --docs notes.pdf research.md
```

| Flag | Purpose |
|---|---|
| `--demo` | Runs the pipeline with fake LLM/search responses |
| `--max-results` | Maximum search results retrieved per sub-question |
| `--max-iterations` | Maximum number of critique/refinement iterations |
| `--docs` | One or more local documents (PDF/Markdown/text) to include as context |
| `-v` | Verbose output |

This list reflects the flags exercised during testing. Refer to `cli.py` directly for the full set of supported options.

## Streamlit UI

```bash
pip install -r requirements-ui.txt
streamlit run app.py
```

The Streamlit interface provides an alternative to the CLI for running the agent interactively.

## Citations

The compiled report references the sources collected during web search. Citations indicate where information was found — they do not guarantee that the cited content is factually correct or complete. Sources should be reviewed directly before relying on any claim in the generated report.

## Testing

```bash
python -m unittest discover -s tests -v
```

or, if pytest is installed:

```bash
pytest tests/
```

## Example

A real local run using Ollama (`llama3.2:3b`) on the topic "The impact of quantum computing on cryptography" was tested with:

```bash
export LLM_PROVIDER=ollama
export LLM_MODEL=llama3.2:3b
python cli.py "The impact of quantum computing on cryptography" --max-results 2 --max-iterations 1 -v
```

This run completed successfully end to end: it planned sub-questions, performed real web searches, synthesized findings, ran a critique pass that identified gaps, and compiled a final Markdown report. This describes the outcome of a single local test run, not a performance benchmark, and figures such as run time or source count are not representative metrics.

## Limitations

- Local, smaller language models may produce weaker synthesis than larger hosted models.
- Web search results can be incomplete, noisy, or occasionally irrelevant.
- Generated content can contain factual errors and should not be treated as authoritative without checking cited sources.
- Research quality depends heavily on both retrieval quality and the LLM being used.
- Local models can run slowly on limited hardware.
- The critique/refinement loop is bounded, not an open-ended autonomous process.
- This is primarily a portfolio and learning project, not a production research platform.
- The agent is not a substitute for expert literature review.

## Future Improvements

The following are potential directions for future work, not currently implemented features:

- Better source ranking and credibility scoring
- Improved retrieval and document chunking
- Richer citation validation
- Structured research memory across sessions
- More rigorous evaluation of report quality
- Asynchronous search execution
- Support for stronger local models as hardware allows
- Persistent research history
- Improved UI

## Portfolio Notes

This project was used to study and practice several concepts relevant to applied AI/ML engineering:

- Designing an agentic workflow as an explicit state graph (LangGraph)
- Managing shared state across multiple workflow steps
- Building an LLM provider abstraction supporting multiple backends (Anthropic, OpenAI, Ollama)
- Implementing tool-calling components for web search and document loading
- Implementing a bounded iterative refinement loop (critique → refine → re-check)
- Working with local LLM deployment via Ollama
- Structuring a Python project with a CLI, an optional UI, and a test suite

This project does not represent original design of the underlying architecture, LangGraph, or the agent pattern itself — see Attribution below.

## Attribution

This project was developed by adapting and studying an existing open-source Research Agent implementation. The codebase was used as a starting point, then configured, tested, and adapted for local execution with Ollama. Further modifications and improvements are being made as part of ongoing work on this repository.

## License

See the [LICENSE](LICENSE) file for license information.