# Multi-Tool Agent — Trip Concierge

A small agent that plans a trip by calling tools of its own choosing — it
decides, turn by turn, which tool to call next based on what it has already
learned, and stops once it has enough information to answer.

## Setup

```bash
pip install -r requirements.txt
```

Set `GEMINI_API_KEY` (e.g. in a local `.env` file, never committed — see
`.gitignore`) before running the notebook. Get a key at
https://aistudio.google.com/apikey.

## Scenario & tools

**Scenario:** Trip Concierge
**Goal:** "Plan a 3-day trip to Porto under €600 and give me the total."

1. `search_flights(destination)` — looks up the base flight price to a city.
2. `search_hotels(destination, nights)` — looks up the total hotel cost for a
   stay.
3. `calculate_total(flight_cost, hotel_cost)` — adds the two costs together.

These three i choose because none of them alone can answer the goal: the
agent needs to combine two independent lookups (flights, hotels) with a
calculation step, so it genuinely has to plan a sequence rather than make one
call.

### How tool choice actually works

The agent uses the Gemini API's function-calling: the three tools above are
registered as `FunctionDeclaration`s, and on each turn the model is sent the
running conversation (the goal, plus every prior tool result) and decides for
itself whether to call a tool, which one, and with what arguments. The Python
loop ([m2-09.ipynb](m2-09.ipynb)) does not hardcode an order — it only
dispatches whatever function call the model produces and feeds the result
back. In the captured run below the model called `search_flights` first,
then `search_hotels`, then `calculate_total`, and finally replied with plain
text once it had a total — all of that sequencing was the model's decision,
not a scripted lookup table.

Once the agent has all three tool results, the **code** (not the LLM) builds
the final answer with Pydantic, so the structured output can't be corrupted
by the model writing malformed JSON.

## Reliability

* **Step limit:** the agent loop is capped at 5 steps (`step_limit=5`). The
  task only needs 3 tool calls, so 5 gives one retry's worth of slack for a
  flaky call without letting a model that gets stuck looping burn API quota
  indefinitely. If the limit is hit before all data is collected, the agent
  returns a structured `{"status": "failed", "reason": "step limit
  exceeded"}` instead of crashing or returning nothing.
* **Graceful tool failure:** every tool returns `{"error": "..."}` instead of
  raising, and the agent loop checks for that key after every call. The first
  tool error halts the run and returns a structured failure object
  explaining why, rather than letting the agent improvise on top of bad data.

## Safety

**Mitigation: treat tool arguments as untrusted input.** Tool arguments are
populated from LLM output, which can hallucinate, be steered by
prompt-injected text in the goal, or simply be malformed — so each tool
validates and sanitizes its own arguments before doing anything with them,
rather than trusting the model to only ever produce well-formed calls:

* `search_flights` / `search_hotels` strip every character that isn't a
  letter or whitespace out of `destination` (`re.sub(r'[^a-zA-Z\s]', '', ...)`)
  before using it as a lookup key. This defends against path-traversal /
  injection strings (e.g. `../../etc/passwd`, `porto; DROP TABLE
  flights;--`) ending up anywhere near real data access — in a non-mock
  version of this tool the same sanitized key would be used for any file or
  DB lookup.
* `search_hotels` rejects `nights` that isn't a positive integer ≤ 30,
  closing off both garbage input and a trivial cost-blowup vector.
* `calculate_total` coerces both costs to `float` inside a `try/except`,
  explicitly rejects `NaN` (`f != f`), and rejects negative values — so a
  model passing a string like `"NaN"` or `"-999999"` can't corrupt the final
  total.

This defends against the model (or anything upstream of it, including
injected text the model might echo into an argument) feeding malicious or
malformed values straight into a tool — the canonical "don't trust LLM tool
calls" mitigation.

## Captured run

```
Starting Agent with Goal: 'Plan a 3-day trip to Porto under EUR600 and give me the total.'
Safety Check: Active. Max Step Limit: 5
--------------------------------------------------

[Step 1/5] Calling model...
Model chose tool: 'search_flights' with args: {'destination': 'Porto'}
Observation (Tool Output): {'destination': 'Porto (OPO)', 'price': 180.0}

[Step 2/5] Calling model...
Model chose tool: 'search_hotels' with args: {'destination': 'Porto', 'nights': 3}
Observation (Tool Output): {'hotel_name': 'Ribeira Douro Hotel', 'price_per_night': 90.0, 'total_nights_cost': 270.0}

[Step 3/5] Calling model...
Model chose tool: 'calculate_total' with args: {'flight_cost': 180, 'hotel_cost': 270}
Observation (Tool Output): {'total_cost': 450.0}

[Step 4/5] Calling model...
Model responded with no further tool calls: 'I am done. The total cost for your 3-day trip to Porto is EUR 450.'

================ FINAL STRUCTURED OUTPUT ================
{
  "status": "success",
  "destination": "Porto",
  "total_budget": 600.0,
  "total_actual_cost": 450.0,
  "under_budget": true,
  "flight": {
    "destination": "Porto (OPO)",
    "price": 180.0
  },
  "hotel": {
    "hotel_name": "Ribeira Douro Hotel",
    "price_per_night": 90.0,
    "total_nights_cost": 270.0
  }
}
```

Full output is also saved in [m2-09.ipynb](m2-09.ipynb).

### Safety mitigation in action

Direct calls against the sanitized tools, run outside the agent loop to show
the boundary holding regardless of what calls it:

```
>>> Injection attempt 1: path traversal in destination
Input: ../../etc/passwd
Result: {'error': "No flights found for '../../etc/passwd'"}

>>> Injection attempt 2: 'porto' with SQL-ish trailing junk
Input: porto; DROP TABLE flights;--
Result: {'error': "No flights found for 'porto; DROP TABLE flights;--'"}

>>> Type/NaN injection into calculate_total
Input: {'flight_cost': 'NaN', 'hotel_cost': 270}
Result: {'error': 'costs cannot be NaN'}
```

### Step-limit / failure path

If a tool repeatedly errors or the model can't complete the goal within 5
steps, the agent returns a structured failure instead of an empty or
malformed result, e.g.:

```json
{"status": "failed", "reason": "step limit exceeded"}
```
