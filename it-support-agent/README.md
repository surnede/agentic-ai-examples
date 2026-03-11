# IT Support Agent

An autonomous Level 1 IT support agent powered by OpenAI GPT-4o-mini. It investigates server incidents by checking health metrics and logs, then decides whether to restart services or escalate to a human engineer.

## How It Works

```
User reports incident
  → Agent checks server health (CPU, memory)
  → Agent fetches recent logs
  → Decision:
      - CPU/Memory > 90% → Restart the service
      - Dependency failure (e.g., connection refused) → Escalate to engineer
      - Everything healthy → Report no issues
```

## Project Structure

```
it-support-agent/
├── README.md                    # This file
├── src/
│   ├── tools.py                 # PART 1: Tool functions (health, logs, restart, escalate)
│   ├── agent.py                 # PART 2: Agent schema, system prompt, execution loop
│   └── main.py                  # PART 3: Entry point — runs test scenarios
└── tests/
    └── test_scenarios.py        # PART 4: Unit tests for tool functions
```

## Setup

1. Install dependencies:

   ```bash
   pip install openai python-dotenv
   ```

2. Create a `.env` file in the `agentic-ai-examples/` root with your API key:
   ```
   OPENAI_API_KEY=sk-your-key-here
   ```

## Run

**Run the agent (all 5 scenarios):**

```bash
cd it-support-agent/src
python main.py
```

**Run unit tests:**

```bash
cd it-support-agent
python tests/test_scenarios.py
```

## Test Scenarios

| Scenario | Server            | Expected Action            |
| -------- | ----------------- | -------------------------- |
| A        | payment-server-01 | Restart (CPU 98%)          |
| B        | db-node-02        | Report healthy             |
| C        | auth-service-03   | Restart (Memory 95%)       |
| D        | search-index-09   | Escalate (dependency down) |
| E        | frontend-node-04  | Report healthy             |

## Tools

| Tool                   | Description                                       |
| ---------------------- | ------------------------------------------------- |
| `get_server_health`    | Returns CPU, memory, and status for a server      |
| `fetch_recent_logs`    | Returns the last N log lines from a server        |
| `restart_service`      | Restarts a server service                         |
| `escalate_to_engineer` | Creates an escalation ticket for a human engineer |

Sequence Diagram:
sequenceDiagram
autonumber
participant M as main.py
participant A as agent.py<br/>run_it_agent()
participant API as OpenAI API<br/>(gpt-4o-mini)
participant T as tools.py<br/>AVAILABLE_FUNCTIONS

    Note over M: python main.py

    M->>A: run_it_agent("payment-server-01 is extremely slow and timing out")

    Note over A: messages = [<br/>  {role: "system", content: SYSTEM_PROMPT},<br/>  {role: "user", content: user_issue}<br/>]

    rect rgb(50, 50, 100)
        Note right of A: LOOP 1 — Investigate
        A->>API: client.chat.completions.create(<br/>  model="gpt-4o-mini",<br/>  messages=[system, user],<br/>  tools=TOOLS_SCHEMA,<br/>  tool_choice="auto"<br/>)
        API-->>A: response.choices[0].message<br/>tool_calls: [{name: "get_server_health",<br/>args: {server_id: "payment-server-01"}}]
        Note over A: response_msg = response.choices[0].message<br/>messages.append(response_msg)
        Note over A: func_name = "get_server_health"<br/>func_args = {"server_id": "payment-server-01"}
        A->>A: AVAILABLE_FUNCTIONS.get("get_server_health")
        A->>T: get_server_health(server_id="payment-server-01")
        T-->>A: '{"cpu": "98%", "memory": "40%", "status": "Warning"}'
        Note over A: messages.append({<br/>  role: "tool",<br/>  tool_call_id: "call_001",<br/>  name: "get_server_health",<br/>  content: '{"cpu":"98%"...}'<br/>})
    end

    rect rgb(50, 80, 50)
        Note right of A: LOOP 2 — Diagnose
        A->>API: client.chat.completions.create(<br/>  messages=[system, user,<br/>  assistant_tool_call, tool_result]<br/>)
        Note over API: LLM sees CPU=98%<br/>"CPU>90% but is it dependency error?"<br/>"Need logs to decide Rule 2 vs Rule 3"
        API-->>A: tool_calls: [{name: "fetch_recent_logs",<br/>args: {server_id: "payment-server-01"}}]
        Note over A: func_name = "fetch_recent_logs"<br/>func_args = {"server_id": "payment-server-01"}
        A->>A: AVAILABLE_FUNCTIONS.get("fetch_recent_logs")
        A->>T: fetch_recent_logs(server_id="payment-server-01", lines=5)
        T-->>A: '{"logs": ["CPU threshold exceeded 90%",<br/>"Thread pool exhaustion",<br/>"Process hung, not accepting connections",<br/>"Timeout waiting for thread"]}'
        Note over A: messages.append({role: "tool", content: logs})
    end

    rect rgb(100, 50, 50)
        Note right of A: LOOP 3 — Act
        A->>API: client.chat.completions.create(<br/>  messages=[system, user,<br/>  ...health_call, health_result,<br/>  ...logs_call, logs_result]<br/>)
        Note over API: LLM sees:<br/>CPU=98% > 90% (Rule 2 matches)<br/>Logs: CPU issues, NOT "connection refused"<br/>(Rule 3 does NOT match)<br/>Decision: RESTART
        API-->>A: tool_calls: [{name: "restart_service",<br/>args: {server_id: "payment-server-01"}}]
        A->>A: AVAILABLE_FUNCTIONS.get("restart_service")
        A->>T: restart_service(server_id="payment-server-01")
        T-->>A: '{"status": "success",<br/>"message": "Server payment-server-01 restarted"}'
        Note over A: messages.append({role: "tool", content: restart_result})
    end

    rect rgb(80, 80, 30)
        Note right of A: LOOP 4 — Final Answer
        A->>API: client.chat.completions.create(<br/>  messages=[system, user,<br/>  ...all previous tool calls and results]<br/>)
        Note over API: LLM: "Investigation complete.<br/>Health checked. Logs checked. Restarted.<br/>No more tools needed."
        API-->>A: tool_calls: None<br/>content: "I investigated payment-server-01.<br/>CPU was at 98%. Logs showed thread pool<br/>exhaustion. Restarted successfully."
        Note over A: tool_calls is None → else branch
        A->>M: print("[FINAL RESPONSE]: ...")<br/>break → loop ends
    end
