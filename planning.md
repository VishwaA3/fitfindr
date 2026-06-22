# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Searches the mock listings dataset and returns items that match the user's description, size, and price limit.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): a description of the item the user wants (For ex. "vintage graphic tee").
- `size` (str): clothing size to filter by. If the size is 'None', size is not filtered.
- `max_price` (float): maximum price; if the price is 'None', price is not filtered.

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
A list of listing dictionaries. Each dictionary contains: id, title, description, category, style_tags, size, condition, price, colors, brand, platform. If no matches are found, then an empty list is returned. 

**What happens if it fails or returns nothing:**
<!-- What should the agent do if no listings match? -->
If results is empty then the agent sets session["error"] = "No listings matched your search. Try a broader description, a different size, or a higher price." and returns early before calling suggest_outfit or create_fit_card.

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Given a specific thrifted item and the user's wardrobe, suggest one or more complete outfit combinations using an LLM.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): a single listing dictionary (the item found by search_listings), containing fields like title, category, colors, style_tags.
- `wardrobe` (dict): the user's wardrobe with an "items" key containing a list of wardrobe item dicts, each with name, category, colors, style_tags, notes

**What it returns:**
<!-- Describe the return value -->
A string containing one or more outfit suggestions written in natural language. 

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
If wardrobe["items"] is empty, the tool still runs but prompts the LLM for general styling advice instead of personalized combinations. Instead, it returns a string like "This item pairs well with wide-leg pants and chunky sneakers — classic streetwear styling." Does not raises an exception.

---

### Tool 3: create_fit_card

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Generates a short, caption-style description of a complete outfit. Something like the kind of thing someone would post on Instagram. Uses an LLM to produce something different each time.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (str): the outfit suggestion string returned by suggest_outfit.
- `new_item` (dict): the listing dictionary for the thrifted item, used to access details like title, price, and platform of that thrifted item. 

**What it returns:**
<!-- Describe the return value -->
A string of a short and casual caption. 

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the outfit data is incomplete? -->
If outfit is an empty string, return the error message string noting how the outfit description was missing. Does not raise an exception.

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->
1. First, call search_listings. Pass in description, size, max_price.
2. If results == [], set session["error"] and return session early and stop. 
3. If results is not empty, set session["selected_item"] = results[0].
4. Call suggest_outfit. Pass in session["selected_item"], wardrobe.
5. Store the returned string in session["outfit_suggestion"].
6. Call create_fit_card. Pass in session["outfit_suggestion"], session["selected_item"].
7. Store the returned string in session["fit_card"].
8. Return session.

---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->
A dictionary is initialized at the start of each run_agent() call to track the following sessions:
- session["selected_item"]: the listing dictionary returned by search_listings 
- session["outfit_suggestion"]: the string returned by suggest_outfit 
- session["fit_card"]: the string returned by create_fit_card
- session["error"]: an error message string, set if any tool fails or returns nothing

Each tool receives its inputs directly from the session dictionary.

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Sets session["error"] = "No listings matched your search. Try a broader description, a different size, or a higher price." Stops and returns early. |
| suggest_outfit | Wardrobe is empty | LLM is prompted for general styling advice instead. Returns a useful string, never crashes. |
| create_fit_card | Outfit input is empty string | Returns "Could not generate a fit card — outfit description was missing." Never raises an exception. |

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     Use ASCII art or a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html).
     Do NOT embed an image — graders need to read your diagram directly in the file;
     an embedded image or screenshot cannot be evaluated.
     You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->

User query (description, size, max_price, wardrobe)
    │
    ▼
Planning Loop
    │
    ├─► search_listings(description, size, max_price)
    │       │ results == []
    │       ├──► session["error"] = "No listings matched..." → return session
    │       │
    │       │ results != []
    │       ▼
    │   session["selected_item"] = results[0]
    │       │
    ├─► suggest_outfit(session["selected_item"], wardrobe)
    │       │ wardrobe["items"] == []
    │       ├──► LLM gives general advice (no crash)
    │       │
    │       ▼
    │   session["outfit_suggestion"] = "..."
    │       │
    └─► create_fit_card(session["outfit_suggestion"], session["selected_item"])
            │ outfit == ""
            ├──► return "Could not generate a fit card..."
            │
            ▼
        session["fit_card"] = "..."
            │
            ▼
        Return session

---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**
For search_listings, I'll give Claude the Tool 1 spec from planning.md (inputs, return value, failure mode) and ask it to implement the function using load_listings() from utils/data_loader.py. I'll verify the generated code filters by all three parameters and handles the empty-results case before running it. Then I'll test it with 3 queries: one that returns results, one with no matches, and one with a price filter.
For suggest_outfit, I'll give Claude the Tool 2 spec and ask it to implement the function using the Groq API with llama-3.3-70b-versatile. I'll verify it handles the empty wardrobe case before running. I'll test it with get_example_wardrobe() and get_empty_wardrobe() separately.
For create_fit_card, I'll give Claude the Tool 3 spec and ask it to implement the function using the Groq API. I'll verify it guards against an empty outfit string. I'll run it 3 times on the same input to confirm the outputs vary.

**Milestone 4 — Planning loop and state management:**
I'll give Claude the Planning Loop section, State Management section, and Architecture diagram from planning.md and ask it to implement run_agent() in agent.py. I'll verify the generated code branches on the search_listings result, stores values in the session dictionary, and does not call all three tools unconditionally. Then I'll test the no-results path by running an impossible query and confirming session["fit_card"] is None.

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!-- What does the agent do first? Which tool is called? With what input? -->
The agent calls search_listings("vintage graphic tee", size=None, max_price=30.0). It returns a list of matching listings. The agent picks results[0] and stores it in session["selected_item"].

**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
The agent calls suggest_outfit(new_item=session["selected_item"], wardrobe=get_example_wardrobe()).
The LLM returns a styling suggestion based on the tee and the wardrobe items.
Result is stored in session["outfit_suggestion"].

**Step 3:**
<!-- Continue until the full interaction is complete -->
The agent calls create_fit_card(outfit=session["outfit_suggestion"], new_item=session["selected_item"]).
The LLM returns a short, caption-style fit card.
Result stored in session["fit_card"].

**Final output to user:**
<!-- What does the user actually see at the end? -->
All three panels populate: the matched listing, the outfit suggestion, and the fit card caption.

If search_listings returns [], the agent sets session["error"] and returns early.
suggest_outfit and create_fit_card are never called.
The user sees: "No listings matched your search. Try a broader description or higher price."