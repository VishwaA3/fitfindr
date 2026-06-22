# FitFindr — Starter Kit

This starter kit contains everything you need to begin Project 2.

## What's Included

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── planning.md                # Your planning template — fill this out first
└── requirements.txt           # Python dependencies
```

## Setup

**macOS / Linux:**
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Windows:**
```bash
python -m venv .venv
source .venv/Scripts/activate
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
```
GROQ_API_KEY=your_key_here
```

## The Mock Listings Dataset

`data/listings.json` contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`.

Load it with:
```python
from utils.data_loader import load_listings
listings = load_listings()
```

## The Wardrobe Schema

`data/wardrobe_schema.json` defines the format your agent uses to represent a user's existing wardrobe. It includes:

- `schema`: field definitions for a wardrobe item
- `example_wardrobe`: a sample wardrobe with 10 items you can use for testing
- `empty_wardrobe`: a starting template for a new user

Load an example wardrobe with:
```python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()
```

## Tool Inventory

Your README submission must document each tool's name, inputs, and return value. **These must exactly match your actual function signatures in `tools.py`.** Your documented interfaces will be checked against your actual function signatures in `tools.py` — if the parameter count or types contradict what's in the code, you may not receive full credit for that tool.

**search_listings(description: str, size: str | None, max_price: float | None) → list[dict]**
Searches the listings dataset by keyword overlap against title, description, category, style_tags, and brand. Filters by size and price if provided. Returns a list of matching listing dicts sorted by relevance score. Returns an empty list if nothing matches.

**suggest_outfit(new_item: dict, wardrobe: dict) → str**
Takes the selected listing and the user's wardrobe and calls the Groq LLM to suggest 1-2 outfit combinations. If the wardrobe is empty, prompts the LLM for general styling advice instead. Always returns a non-empty string.

**create_fit_card(outfit: str, new_item: dict) → str**
Takes the outfit suggestion and listing details and calls the Groq LLM to generate a short, casual Instagram-style caption. Guards against an empty outfit string and returns an error message instead of crashing.

---

## Interaction Walkthrough

<!-- Walk through a complete interaction step by step: natural language query → each tool call (and why) → final fit card.
     Walk through this carefully — it's how graders follow your agent's reasoning without a live demo.
     Use a specific example — do not leave this as a template. -->

**User query:** "90s track jacket in size M"

**Step 1 — Tool called:**
- Tool: search_listings
- Input: description="90s track jacket", size="M", max_price=None
- Why this tool: always called first to find a matching listing before anything else can run
- Output: matching listings; top result is "90s Track Jacket — Navy/White Stripe", $45, poshmark, size M, condition excellent

**Step 2 — Tool called:**
- Tool: suggest_outfit
- Input: new_item=session["selected_item"] (90s Track Jacket dict), wardrobe=get_example_wardrobe()
- Why this tool: search returned a result, so the agent proceeds to generate outfit ideas using the user's wardrobe
- Output: suggested pairing the jacket with baggy straight-leg jeans and chunky white sneakers, or wide-leg khaki trousers and black combat boots for a more edgy look

**Step 3 — Tool called:**
- Tool: create_fit_card
- Input: outfit=session["outfit_suggestion"], new_item=session["selected_item"]
- Why this tool: outfit suggestion was generated successfully, so the agent creates a shareable caption
- Output: "just scored this sick 90s track jacket for $45 on poshmark and i'm obsessed. paired it with my fave baggy jeans and chunky white sneakers for a laid-back vibe 🤙"

**Final output to user:**
All three panels are displayed in the Gradio UI: the listing details, the outfit suggestion, and the fit card caption.

---

## Error Handling and Fail Points

<!-- For each tool, describe the specific failure mode and what your agent does in response.
     This maps to the error handling section of the rubric (F5-C1). -->

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| `search_listings` | No results match the query | Sets session["error"] = "No listings matched your search. Try a broader description, a different size, or a higher price." Returns early without calling other tools. Tested with "designer ballgown size XXS under $5" — returned [] with no exception. |
| `suggest_outfit` | Wardrobe is empty | Prompts the LLM for general styling advice instead of personalized combinations. Tested with get_empty_wardrobe() — returned useful styling advice without crashing. |
| `create_fit_card` | outfit is empty string | Returns "Could not generate a fit card — outfit description was missing." without raising an exception. Tested by calling create_fit_card("", results[0]) directly. |

---

## Spec Reflection

<!-- Answer both questions with at least 2–3 sentences each. -->

**One way planning.md helped during implementation:**
Writing the planning loop as numbered conditional steps in planning.md made the agent.py implementation straightforward. I knew exactly what to check at each step, specifically that search_listings had to return results before suggest_outfit could be called, and so there was no ambiguity when writing the code.

**One divergence from your spec, and why:**
The spec described the session dict and planning loop in detail but didn't specify how the query would be parsed into description, size, and max_price. I had to figure out the regex patterns during implementation. In hindsight I should have planned the parsing step more explicitly in planning.md before coding it.

---

## Where to Start

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.

---

## AI Usage

**Instance 1 — search_listings implementation:**
I gave Claude the Tool 1 spec from planning.md (inputs, return value, failure mode) and asked it to implement the function using load_listings(). The generated code scored listings by keyword overlap but used item["brand"] directly, which caused a TypeError when brand was None for some listings. I fixed it by changing item["brand"] to item["brand"] or "".

**Instance 2 — planning loop implementation:**
I gave Claude the Planning Loop section, State Management section, and Architecture diagram from planning.md and asked it to implement run_agent() in agent.py. The generated code matched the spec correctly. I verified both the happy path and no-results path worked before committing.
