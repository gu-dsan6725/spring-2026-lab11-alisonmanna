# Task 3 Observations: A2A Communication and Agent Discovery

### 1. A2A Messages Exchanged Between Agents

This was the JSON-RPC request that was logged by the Travel Assistant when it delegated the booking task to the Flight booking agent (from `remote_agent_client.py`):

**Request from Travel Assistant -> Flight Booking Agent:**
```json
{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "I need to book flight ID 1. Please reserve 2 seats, confirm the reservation, and process the payment."
        }
      ],
      "messageId": "436e8b585beb46199fad5aa83757d488"
    }
  }
}
```

The Flight Booking Agent received this at 22:30:07 & processed the request using its own LLM + tools (check_availability, reserve_flight, confirm_booking, process_payment). It returned the response as a streaming Task/Message A2A object that was parsed by `RemoteAgentClient.send_message()`.

Full sequence:
1. 22:30:04 — LiteLLM call: Travel Assistant LLM reasons about booking request, decides to discover agents
2. 22:30:07 — `invoke_remote_agent(agent_id='/flight-booking-agent', ...)` tool called
3. 2:30:07 — A2A REQUEST logged and sent via HTTP POST to `http://127.0.0.1:10002/`
4. 22:30:11 — "Successfully invoked Flight Booking Agent" logged after response received
5. 22:30:11 — LiteLLM call: Travel Assistant LLM collects final response for user

---

### 2. How Travel Assistant Discovered Flight Booking Agent

Discovery was handled by `discover_remote_agents` tool in the `agent.py`. When the Travel Assistant LLM determined it needed the ability to make bookings, it called this tool with a query. After that:

- Called `RegistryDiscoveryClient.discover_by_semantic_search(query=..., max_results=5)` in `registry_discovery_client.py`
- Sent HTTP POST to `http://127.0.0.1:7861/api/agents/discover/semantic?query=<query>`
- Registry stub returned a hardcoded response containing the Flight Booking Agent
- Agent metadata was  stored in `RemoteAgentCache` under `/flight-booking-agent` (the agent `path` field)
- On next LLM step, agent called `invoke_remote_agent(agent_id='/flight-booking-agent', message=...)`, which looked up `RemoteAgentClient` and called `send_message()`

Before sending the message `RemoteAgentClient._ensure_initialized()` performed another step of fetching the flight booking agent's card from using `A2ACardResolver`, then used that card to configure the A2A client.

---

### 3. JSON-RPC Request/Response Format

**Request format**:
```json
{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{ "kind": "text", "text": "<message content>" }],
      "messageId": "<uuid hex>"
    }
  }
}
```

**Response format** (parsed by `RemoteAgentClient.send_message()` and by the test's `extract_response_text()`):
```json
{
  "result": {
    "artifacts": [
      {
        "parts": [{ "text": "<agent response text>" }]
      }
    ]
  }
}
```

Both the agents use the protocol version `0.3.0` with `JSONRPC` as the transport. The method is always `message/send`.

---

### 4. Agent Card Info & How it was Used

Both agent cards were retrieved from `/.well-known/agent-card.json` and contain:

| Field | Purpose |
|---|---|
| `name` / `description` | Identity (human readable); for registry to match discovery queries |
| `url` | Where to send A2A messages to (e.g., `http://127.0.0.1:10002/`) |
| `skills` | List of capability IDs, names, descriptions the agent exposes |
| `protocolVersion` | `"0.3.0"`, make sure both agents speak the same A2A version |
| `preferredTransport` | `"JSONRPC"`, for telling the calling agent how to format the requests |
| `capabilities.streaming` | `true`, A2A client uses streaming to receive the response |

**Travel Assistant agent card skills**: `search_flights`, `check_prices`, `get_recommendations`, `create_trip_plan`, `discover_remote_agents`, `view_cached_remote_agents`, `invoke_remote_agent`

**Flight Booking Agent card skills**: `check_availability`, `reserve_flight`, `confirm_booking`, `process_payment`, `manage_reservation`

How it was use in the workflow:
- The registry returned a partial agent record (name, url, path, skills, tags)
- `RemoteAgentClient` fetched full agent card from the flightbooking agent directly via `A2ACardResolver` to configure the A2A client with the proper transport and protocol settings before sending any message

---

### 5. Benefits and Limitations

**Benefits:**
- The registry enables loose coupling. The Travel Assistant does not need to know the Flight Booking Agent's URL ahead of time, it just asks the registry at runtime
- Agents advertise what they can do through the agent card, so the Travel Assistant can figure out who to delegate to without anything being hardcoded
- Both agents use the same JSON-RPC message format, which makes it easy to add new agents to the system w/o changing how communication works

**Limitations**:
- The registry is a stub so it always returns the Flight Booking Agent no matter what query is sent, so the "semantic search" isn't real (in producution would need actual search over agent descriptions)
- Discovery adds extra trips back and forth on every request (registry call + fetching agent card) which could get slow if there were many agents 
- Agent cache is stored in memory and never expires -- if an agent's URL changed the travel assistant would keep using the old one until it was restarted
- No authentication between agents, so technically anyone who knew the endpoint URLs could send messages to either agent