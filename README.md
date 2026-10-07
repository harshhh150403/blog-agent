# Blog Writing Agent

An AI agent that writes complete technical blog posts from a single topic. It is built with [LangGraph](https://github.com/langchain-ai/langgraph) and has a [Streamlit](https://streamlit.io) UI. For each topic, it can:

- decide whether the topic needs fresh web research, and run that research with Tavily,
- plan an outline of 5–9 sections,
- write all sections in parallel, with citations when research was used,
- add up to 3 diagrams generated with Gemini,
- save the result as a Markdown file you can preview and download.

## How it works

```
START → router ─┬─ (needs research) → research ─┐
                └──────────────────────────────┴→ orchestrator → worker × N → reducer → END
```

| Node | What it does |
|---|---|
| **router** | Classifies the topic as `closed_book` (evergreen), `hybrid` (evergreen plus recent examples), or `open_book` (news or weekly roundup), and writes the search queries. |
| **research** | Runs the queries with Tavily and condenses the results into evidence items: title, URL, date, snippet. In `open_book` mode it keeps only results from the last 7 days. |
| **orchestrator** | Writes the blog plan: title, audience, tone, and 5–9 section tasks, each with a goal, bullets, and a target word count. |
| **worker** | Runs once per section, in parallel. Each worker writes one Markdown section and cites evidence URLs where required. |
| **reducer** | A subgraph of three steps: `merge_content` puts the sections together in order, `decide_images` picks where diagrams belong and writes their prompts, and `generate_and_place_images` creates the images and inserts them. |

Models: `gpt-4.1-mini` for all text, `gemini-2.5-flash-image` for images.

## Project structure

```
.
├── bwa_backend.py     # LangGraph pipeline (schemas, nodes, graph)
├── bwa_frontend.py    # Streamlit UI
├── requirements.txt
├── .env               # API keys (git-ignored)
└── README.md
```

Each run creates these files (all git-ignored):

- `<blog_title_slug>.md`: the finished blog, saved in the project root
- `images/`: the generated diagrams

## Setup

Requires Python 3.10 or newer. The virtual environment in this repo uses Python 3.12.

```bash
# 1. Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt
```

### API keys

Fill in `.env` in the project root:

```env
OPENAI_API_KEY=""
TAVILY_API_KEY=""
GOOGLE_API_KEY=""
```

| Variable | Required | Used for | If it's missing |
|---|---|---|---|
| `OPENAI_API_KEY` | Yes | All LLM calls | The app fails to start |
| `TAVILY_API_KEY` | No | Web research | Research returns no evidence, so the post is written from the model's own knowledge |
| `GOOGLE_API_KEY` | No | Image generation | Each image slot is replaced with a note showing the prompt and the error |

Get the keys here: [OpenAI](https://platform.openai.com/api-keys) · [Tavily](https://app.tavily.com) · [Google AI Studio](https://aistudio.google.com/apikey)

## Usage

### Streamlit app

```bash
streamlit run bwa_frontend.py
```

Enter a topic in the sidebar, set the **As-of date** (research and recency filtering use it), and click **Generate Blog**. The tabs show:

- **Plan**: the outline and the settings for each section
- **Evidence**: the sources found during research
- **Markdown Preview**: the rendered blog, with download buttons for the `.md` file or a zip with the images
- **Images**: the image plan and the generated files
- **Logs**: what each graph step returned

Earlier blogs (the `.md` files in the project root) are listed under **Past blogs** in the sidebar, where you can open them again.

### From Python

```python
from datetime import date
from bwa_backend import app

result = app.invoke({
    "topic": "How vector databases work",
    "as_of": date.today().isoformat(),
})

print(result["final"])  # also saved to <slug>.md
```
