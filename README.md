# FitFindr

## Project Overview

FitFindr is a multi-tool AI agent that helps users discover secondhand clothing items and generate outfit ideas using their existing wardrobe. The agent uses a planning loop to decide which tool to call next, stores information in a session state object, and handles failures gracefully when tools return no results.

The system combines three tools:

1. Search for matching thrift listings.
2. Generate outfit suggestions.
3. Create a shareable social-media-style fit card.

The agent changes its behavior based on the output of previous tools instead of running every tool unconditionally.
---
# Tool Inventory

## Tool 1: search_listings

### Purpose

Searches the mock thrift listings dataset and finds items that match the user's description, size, and budget.

### Inputs

- description (str): Keywords describing the item the user wants.
- size (str | None): Desired clothing size.
- max_price (float | None): Maximum budget.

### Output

Returns a list of listing dictionaries.

Each listing may contain:

- id
- title
- description
- category
- style_tags
- size
- condition
- price
- colors
- brand
- platform

### Failure Handling

If no matching listings are found, the function returns:

[] 

instead of raising an exception.
---

## Tool 2: suggest_outfit

### Purpose

Generates outfit suggestions using the selected thrift item and the user's wardrobe.

### Inputs

- new_item (dict): Selected listing returned by search_listings.
- wardrobe (dict): User wardrobe data.

### Output

Returns a string containing one or more outfit suggestions.

### Failure Handling

If the wardrobe is empty, the tool still generates general styling advice rather than crashing or returning an empty response.

Example:

Instead of failing, the tool suggests:

> Pair the graphic tee with high-waisted denim and white sneakers for a casual vintage look.
---

## Tool 3: create_fit_card

### Purpose

Creates a short social-media-style caption based on the selected item and outfit suggestion.

### Inputs

- outfit (str): Outfit suggestion from Tool 2.
- new_item (dict): Selected thrift listing.

### Output

Returns a short caption suitable for Instagram or TikTok.

### Failure Handling
If the outfit string is empty, the function returns an informative error message:

text Fit card could not be created because the outfit suggestion is empty. 

instead of crashing.
---
# Planning Loop

The planning loop is implemented in run_agent().

The workflow is:

### Step 1

Initialize a new session object.

### Step 2

Parse the user query and extract:

- description
- size
- maximum price

Store the parsed values in:

python session["parsed"] 

### Step 3

Call:

python search_listings(description, size, max_price) 

Store results in:

python session["search_results"] 

### Step 4

Check if results are empty.

If:

python results == [] 

the agent:

- sets session["error"]
- returns early
- does not call Tool 2
- does not call Tool 3

### Step 5

Select the best matching listing:

python session["selected_item"] = results[0] 

### Step 6

Call:

python suggest_outfit(selected_item, wardrobe) 

Store the result in:

python session["outfit_suggestion"] 

### Step 7

Call:

python create_fit_card(outfit_suggestion, selected_item) 

Store the result in:

python session["fit_card"] 

### Step 8

Return the completed session.

This planning loop ensures that later tools are only executed if earlier tools succeed.
---
# State Management
The agent stores information in a session dictionary throughout the interaction.

Important session fields include:

python {     "query": ...,     "parsed": ...,     "search_results": ...,     "selected_item": ...,     "wardrobe": ...,     "outfit_suggestion": ...,     "fit_card": ...,     "error": ... } 

State flows between tools as follows:

text search_listings()         ↓ selected_item         ↓ suggest_outfit()         ↓ outfit_suggestion         ↓ create_fit_card()         ↓ fit_card 

The user does not need to re-enter information because each tool uses values stored in the session.
---
# Error Handling

## Tool 1 Failure Example

Input:

text designer ballgown size XXS under $5 

Output:

text No matching listings were found. Try a broader description, different size, or higher budget. 

The agent stops immediately.

Values observed during testing:

python session["error"] != None session["fit_card"] == None 
---
## Tool 2 Failure Example

Tested with:

python get_empty_wardrobe() 

Instead of crashing, the tool returned a general styling recommendation.
---
## Tool 3 Failure Example

Input:

python create_fit_card("", item) 

Output:

text Fit card could not be created because the outfit suggestion is empty. 
---

# Example Interaction

User Query:

text vintage graphic tee under $30 

### Tool 1

Called:

python search_listings(     description="vintage graphic tee",     size=None,     max_price=30 ) 

Returned:

python [     {         "title": "Y2K Baby Tee — Butterfly Print",         ...     } ] 

### Tool 2

Called:

python suggest_outfit(selected_item, wardrobe) 

Returned:

text Pair the tee with baggy straight-leg jeans and chunky white sneakers. 

### Tool 3

Called:

python create_fit_card(outfit_suggestion, selected_item) 

Returned:

text Just scored the cutest Y2K Baby Tee on Depop for $18... 

Final user output contains:

- Listing information
- Outfit recommendation
- Social media fit card

---
# Spec Reflection

The implementation closely follows the original design created in planning.md.

The final system successfully:

- Uses three independent tools
- Maintains state across tool calls
- Uses a planning loop with branching logic
- Stops execution when errors occur
- Handles empty results gracefully
- Produces different behavior depending on tool outputs

One improvement that could be added in the future is retry logic that automatically broadens search constraints when no listings are found.
---

# AI Usage

## Example 1 — Tool Implementation

I used ChatGPT to help implement the three required tools.

### Input Provided

I supplied:

- Tool specifications from planning.md
- Function signatures from tools.py
- Required inputs, outputs, and failure modes

### Output Produced

ChatGPT generated initial implementations for:

- search_listings()
- suggest_outfit()
- create_fit_card()

### Verification

Before accepting the code, I:

- tested each tool individually
- confirmed correct return types
- verified failure cases
- modified portions of the generated code to match my planned behavior

---

## Example 2 — Planning Loop Implementation

I used ChatGPT to help implement run_agent().

### Input Provided

I supplied:

- Planning Loop section from planning.md
- State Management section from planning.md
- Mermaid architecture diagram

### Output Produced

ChatGPT generated an initial planning loop implementation that:

- parsed user queries
- called tools in sequence
- stored state in a session dictionary

### Verification

I verified that:

- selected_item flowed into suggest_outfit
- outfit_suggestion flowed into create_fit_card
- the no-results branch returned early
- fit_card remained None when search failed

Only after testing these conditions did I keep the implementation.

---

# Testing

Run tests:

bash PYTHONPATH=. pytest tests/ 

Current Results:

text 5 passed 

Test coverage includes:

- successful search
- empty search results
- price filtering
- empty wardrobe handling
- empty fit card handling