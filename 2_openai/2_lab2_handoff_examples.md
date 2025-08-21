## Handoff examples — structured protocol, validation, and wrappers

This file contains short, copy-pasteable examples you can add to the end of `2_lab2.ipynb` to demonstrate a structured JSON handoff protocol, a Sales Manager instruction snippet, and two server-side wrapper approaches that *force* correct behavior (ignore/validate manager-provided input).

- **Goal**: show a reliable pattern where the Sales Manager delegates control to a specialized `Email Manager` via a single-line JSON handoff. The receiver has permission to send email and is the only agent with that capability.

---

### 1) Recommended handoff protocol (single-line JSON)

Instruction for the Sales Manager (paste into its `instructions`):

```text
When handing off to the Email Manager, output _only_ a single-line JSON object with this exact shape and no surrounding text or explanation:
{"handoff":"Email Manager","payload":{"body":"<exact chosen email body>"},"correlation_id":"<uuid>"}

Example (manager must emit exactly):
{"handoff":"Email Manager","payload":{"body":"Dear CEO..."},"correlation_id":"cid-123"}
```

Why single-line JSON? it's trivial to parse, validate, and reject if malformed.

---

### 2) Server-side wrapper that IGNORES manager-provided input and uses canonical user input

Use this pattern if you want to prevent the manager from inventing or mutating the child-agent inputs.

```python
from agents import function_tool, Runner

# Keep a canonical `user_message` variable in your notebook/session
user_message = "Send out a cold sales email addressed to Dear CEO from Alice"

@function_tool
def sales_agent_fixed_input(index: int) -> dict:
    """Ignore any manager-provided `input` and run the sales_agent using the canonical `user_message`.
    Returns the agent output so the manager still sees the draft (if you want that behavior).
    """
    # index -> which sales agent (0,1,2)
    agent = sales_agents[index]
    result = Runner.run(agent, user_message)
    return {"output": result.final_output}

# Then register tools like: tools = [sales_agent_fixed_input_as_tools..., send_email]
```

Notes: this enforces a single source-of-truth for the child agent input. The manager still decides which agents to call but cannot mutate the payload.

---

### 3) Server-side validator wrapper (rejects malformed handoffs)

Use this pattern when you want to allow the manager to craft inputs but still validate/authorize them before performing sensitive work.

```python
from agents import function_tool, Runner
import json

@function_tool
def validated_handoff(handoff_json: str) -> dict:
    """Validate a single-line JSON handoff. Accept only the expected shape and enforce permissions.
    Example expected shape:
    {"handoff":"Email Manager","payload":{"body":"..."},"correlation_id":"cid-..."}
    """
    try:
        data = json.loads(handoff_json)
    except Exception:
        raise ValueError("Handoff must be valid JSON")

    if data.get("handoff") != "Email Manager":
        raise ValueError("Unsupported handoff target")

    payload = data.get("payload") or {}
    body = payload.get("body")
    if not isinstance(body, str) or len(body) < 20:
        raise ValueError("Email body missing or too short")

    # Optionally check permissions / session ownership using correlation_id/session tokens
    # if not authorized(correlation_id): raise PermissionError(...)

    # Run the Email Manager handoff (asynchronous or synchronous depending on your runtime)
    result = Runner.run(emailer_agent, body)
    return {"status": "ok", "sent_preview": result.final_output[:200]}
```

Notes: this preserves manager flexibility but enforces server-side safety checks before doing I/O or accessing secrets.

---

### 4) Alternate design: pre-generate drafts, then ask manager to choose

If you want full control over what child agents see, run the child agents yourself, collect their outputs, then give the manager exactly those drafts to choose from:

```python
drafts = [Runner.run(sales_agent1, user_message).final_output,
          Runner.run(sales_agent2, user_message).final_output,
          Runner.run(sales_agent3, user_message).final_output]

manager_prompt = "Here are three drafts:\n\n" + "\n\n----\n\n".join(drafts) + "\n\nPick the best draft and return it ONLY."
choice = Runner.run(sales_manager, manager_prompt)
# then pass the exact chosen draft to the emailer_agent via a handoff or wrapper
```

This pattern removes the manager's ability to craft child-inputs entirely and is often simplest for strict workflows.

---

### Quick checklist before using handoffs in production
- Use short, unique handoff names and a single-line JSON protocol.
- Validate payloads server-side (lengths, types, permissions).
- Pass IDs when the receiver has DB access rather than full content when possible.
- Restrict tools/secrets to the receiver agent only.
- Add correlation_id/session_id for observability and tracing.

Feel free to copy the code blocks into the notebook as new cells or ask me to insert them directly into `2_lab2.ipynb` if you'd prefer I append them there instead of adding this separate file.


