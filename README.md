# Multi-Tool Agent - Trip Concierge System

An autonomous, bounded, and guarded agent that reasons about its state, selects appropriate tools dynamically to achieve a complex goal, and emits a rigidly structured parseable JSON dataset.

## 🌟 Scenario & Tool Evaluation

**Chosen Scenario:** Trip Concierge
* **Goal:** "Plan a 3-day trip to Porto under €600 and give me the total."

### Chosen Tools
1.  `search_flights`: Fetches destination airport codes and hard base flight expenses. 
2.  `Google Hotels`: Calculates cumulative nightly lodging figures given a requested stay duration.
3.  `calculate_total`: Safely computes absolute totals, handling standard floating-point safety boundaries.

*Why these tools?* An LLM or isolated agent cannot natively predict real-time mock data without database access. To compute absolute pricing metrics against a firm financial constraint, the agent requires clear access to decoupled components (transit, logging, logic calculation) to properly satisfy a multi-step query.

---

## 🛡️ Reliability Engineering Note
* **Safe Step Bounding:** The agent engine wraps its state execution loop within a deterministic `step_limit` (configured at a maximum ceiling of 5 steps). If a broken tool loop or hallucinated argument forces a cyclic condition, the execution terminates immediately instead of entering an infinite, resource-draining loop.
* **Graceful Degraded Handling:** All tools return standardized error payloads `{"error": "message"}` instead of completely breaking execution. If a tool fails, the loop catches the string output, halts downstream operations cleanly, and outputs a valid JSON breakdown explaining why completion failed.

---

## 🔒 Safety Engineering Note
* **Implemented Mitigation:** String Sanitization, Data Sanitization, and Boundary Enforcement.
* **Threat Vectors Defended Against:** 1.  *Prompt Injection / Path Exploits:* The `destination` variable is stripped using a strict regular expression `[^a-zA-Z\s]`. If a rogue system attempt feeds paths like `../../etc/passwd` or query injections to tools, they are instantly cleansed to safe alphabetic parameters.
    2.  *Type Injection:* `calculate_total` enforces float checks on values to prevent `NaN` arithmetic anomalies or unexpected runtime data crashes.

---

## 📸 Captured Runtime Log

```json
🚀 Starting Agent with Goal: 'Plan a 3-day trip to Porto under €600 and give me the total.'
🛡️ Safety Check: Active. Max Step Limit: 5
--------------------------------------------------

🤖 [Step 1/5] Thinking...
💭 Reason: I need to find out the transit costs first. Calling search_flights.
📞 Action: Call tool 'search_flights' with args: {'destination': 'porto'}
📥 Observation (Tool Output): {'destination': 'Porto (OPO)', 'price': 180.0}

🤖 [Step 2/5] Thinking...
💭 Reason: Flight details acquired. Now I need accommodation metrics. Calling search_hotels.
📞 Action: Call tool 'search_hotels' with args: {'destination': 'porto', 'nights': 3}
📥 Observation (Tool Output): {'hotel_name': 'Ribeira Douro Hotel', 'price_per_night': 90.0, 'total_nights_cost': 270.0}

🤖 [Step 3/5] Thinking...
💭 Reason: I have both separate costs. Now aggregating via calculate_total.
📞 Action: Call tool 'calculate_total' with args: {'flight_cost': 180.0, 'hotel_cost': 270.0}
📥 Observation (Tool Output): {'total_cost': 450.0}

🤖 [Step 4/5] Thinking...
🏁 Reasoner evaluates: All necessary steps executed. Constructing response.

================ FINAL STRUCTURED OUTPUT ================
{
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