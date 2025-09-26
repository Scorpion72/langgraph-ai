# CopilotKit × LangGraph --- Interrupt Flow (Human‑in‑the‑Loop)

A concise, copy‑paste‑ready guide to building interrupt‑driven
Human‑in‑the‑Loop (HITL) flows with CopilotKit and LangGraph.

------------------------------------------------------------------------

## What you'll build

An agent flow that **pauses with `interrupt()`**, renders a **custom
approval UI** in your app, and then **resumes** with the user's decision
(approve / reject / edit).

------------------------------------------------------------------------

## Prerequisites

-   A LangGraph agent (Python or JS) with persistence (thread /
    checkpointing) enabled
-   A CopilotKit frontend (React)
-   A CopilotKit backend bridge to your agent (Copilot Runtime +
    LangGraph SDK)

------------------------------------------------------------------------

## Architecture at a glance

    User → Copilot UI (React) ↔ Copilot Runtime ↔ LangGraph Agent
                                    ▲                      |
                                    │  interrupt event     | pause
                                    └─────────────── UI renders

1)  The graph calls `interrupt()` at a decision point.
2)  CopilotKit surfaces an **Interrupt event** to the frontend.
3)  Your React app renders a custom approval component.
4)  User action supplies a **value** back to the graph, which
    **resumes** execution.

------------------------------------------------------------------------

## Backend: add an interrupt point

### Python (LangGraph)

``` python
from langgraph.checkpoint import MemorySaver
from langgraph.graph import START, END, StateGraph
from typing import TypedDict

class S(TypedDict):
    input: str
    plan: str
    approval: str | None

memory = MemorySaver()

async def plan_node(state: S):
    plan = f"Send email to {state['input']}?"
    # Pause here for human review; pass a payload for the UI
    from langgraph.types import interrupt
    approval = interrupt({
        "title": "Approve action",
        "proposed_plan": plan,
        "options": ["approve", "reject", "edit"],
    })
    return {"plan": plan, "approval": approval}

async def act_node(state: S):
    if state["approval"] == "approve":
        # perform the action
        return {"result": "Action executed"}
    return {"result": "Action skipped"}

builder = StateGraph(S)
builder.add_node("plan", plan_node)
builder.add_node("act", act_node)

builder.add_edge(START, "plan")
builder.add_edge("plan", "act")
builder.add_edge("act", END)

graph = builder.compile(checkpointer=memory)
```

> Notes - `interrupt(payload)` pauses the graph and yields `payload` to
> the UI. - When the UI resolves, the **return value** of
> `interrupt(...)` becomes `approval` (e.g., `"approve"`). - Use a
> persistent checkpointer so the run can idle while waiting for the
> user.

### JavaScript/TypeScript (LangGraph)

``` ts
import { StateGraph, START, END } from "@langgraph/sdk";
import type { State } from "@langgraph/sdk";

interface S extends State {
  input: string;
  plan?: string;
  approval?: string;
}

const plan = async (s: S) => {
  const plan = `Send email to ${s.input}?`;
  // @ts-ignore – pseudo import for clarity
  const { interrupt } = await import("@langgraph/sdk/interrupt");
  const approval = interrupt({
    title: "Approve action",
    proposed_plan: plan,
    options: ["approve", "reject", "edit"],
  });
  return { plan, approval };
};

const act = async (s: S) => {
  if (s.approval === "approve") {
    return { result: "Action executed" };
  }
  return { result: "Action skipped" };
};

const g = new StateGraph<S>()
  .addNode("plan", plan)
  .addNode("act", act)
  .addEdge(START, "plan")
  .addEdge("plan", "act")
  .addEdge("act", END)
  .compile({ /* checkpointer */ });
```

------------------------------------------------------------------------

## Frontend: render the Interrupt UI

### 1) Wire the agent to your UI

Make sure your CopilotKit chat (or headless UI) is connected to the same
**thread / run id** you use for the graph.

### 2) Handle the interrupt event with a hook

Use the `useLangGraphInterrupt` hook to show custom UI when the backend
calls `interrupt(...)`.

``` tsx
import { useLangGraphInterrupt } from "@copilotkit/react";

export function ApprovalSurface() {
  useLangGraphInterrupt({
    onInterrupt: async (evt, { resolve }) => {
      // evt.value is the payload you passed to interrupt(...)
      const { title, proposed_plan } = evt.value ?? {};

      // Show your own modal / drawer / panel here
      const decision = await openApprovalModal({ title, proposed_plan });

      // Pass the decision back to the graph to resume execution
      resolve(decision); // e.g., "approve" | "reject" | { edited: "..." }
    },
  });
  return null; // purely behavioral hook
}
```

> Tips - `resolve(value)` **must** be called to resume the graph. - You
> can return primitives ("approve") or rich objects (e.g.,
> `{ editedPlan: string }`). - Keep the UI idempotent; the hook may
> re-render.

------------------------------------------------------------------------

## Optional: Generative UI (nice UX)

Render a structured card inside chat during the interrupt:

``` tsx
import { useCopilotAction } from "@copilotkit/react";

useCopilotAction({
  name: "show_plan_card",
  parameters: { plan: "string" },
  render: ({ plan }, { resolve }) => (
    <div className="rounded-2xl p-4 shadow">
      <h3 className="font-semibold mb-2">Proposed plan</h3>
      <p className="mb-4">{plan}</p>
      <div className="flex gap-2">
        <button onClick={() => resolve("approve")}>Approve</button>
        <button onClick={() => resolve("reject")}>Reject</button>
      </div>
    </div>
  ),
});
```

------------------------------------------------------------------------

## Common patterns

-   **Tool call approval**: interrupt before executing tools with
    side‑effects.
-   **Data confirmation**: present parsed entities → allow edit → resume
    with sanitized values.
-   **Multi‑step review**: nested or repeated interrupts, each encoding
    a step in a wizard.
-   **Guardrails**: interrupt on policy triggers to require explicit
    user consent.

------------------------------------------------------------------------

## Persistence & threading

-   Enable a **checkpointer** so runs can pause indefinitely.
-   Use a stable **thread / run id** to reconnect UI and backend after
    page reloads.
-   On resume, prefer **append‑only state updates** to avoid clobbering
    prior values.

------------------------------------------------------------------------

## Troubleshooting

-   **Graph doesn't resume** → Ensure the UI calls `resolve(...)`
    exactly once per interrupt.
-   **Old UI stays on screen** → Unmount / close your modal after
    `resolve` and reconcile state.
-   **State looks stale after resume** → Verify your checkpointer and
    that you're reading the latest state on the next node.
-   **Multiple UI renders** → Debounce or guard against duplicate
    `onInterrupt` handlers.

------------------------------------------------------------------------

## Minimal E2E checklist

-   [ ] Graph calls `interrupt(payload)` at the right node
-   [ ] Persistence is configured (checkpoint / thread id)
-   [ ] Frontend listens with `useLangGraphInterrupt`
-   [ ] UI calls `resolve(value)` to resume
-   [ ] Next node consumes the returned value

------------------------------------------------------------------------

## Further reading

-   LangGraph --- Human‑in‑the‑Loop concepts & how‑tos
-   CopilotKit --- `useLangGraphInterrupt` hook, Generative UI,
    LangGraph SDK (JS/Python)

------------------------------------------------------------------------

### License & attribution

This doc is a derivative summary intended for internal use, based on the
CopilotKit + LangGraph interrupt flow documentation and related
references.
