# CopilotKit × LangGraph --- Node Flow (Human‑in‑the‑Loop)

A concise, copy‑paste‑ready guide to building Human‑in‑the‑Loop (HITL)
flows in LangGraph where **entire nodes** are delegated to the user
through CopilotKit.

------------------------------------------------------------------------

## What you'll build

An agent flow where one or more **graph nodes** are resolved **by the
user** instead of the model. The frontend provides a UI, the user
completes the task, and their response flows back into the graph.

------------------------------------------------------------------------

## Prerequisites

-   A LangGraph agent (Python or JS) with persistence (checkpointing)
    enabled
-   A CopilotKit frontend (React)
-   A CopilotKit backend bridge (Copilot Runtime + LangGraph SDK)

------------------------------------------------------------------------

## Architecture at a glance

    User ↔ Copilot UI (React) ↔ Copilot Runtime ↔ LangGraph Agent
                                   │                  ▲
                                   │    node event    │
                                   └───────delegate───┘

1)  The graph reaches a node marked as **human‑in‑the‑loop**.
2)  That node emits a structured payload.
3)  CopilotKit surfaces it to the frontend.
4)  The user resolves the node by providing a value.
5)  The graph resumes with that value as the node output.

------------------------------------------------------------------------

## Backend: define a HITL node

### Python (LangGraph)

``` python
from langgraph.graph import START, END, StateGraph
from langgraph.checkpoint import MemorySaver
from langgraph.types import HumanNode
from typing import TypedDict

class S(TypedDict):
    input: str
    summary: str

memory = MemorySaver()

async def model_node(state: S):
    return {"input": state["input"]}

# A HumanNode is resolved externally (UI)
user_node = HumanNode(
    name="summary_node",
    description="User provides a summary of the input text",
    schema={"summary": "string"}
)

builder = StateGraph(S)
builder.add_node("model", model_node)
builder.add_node("summary", user_node)

builder.add_edge(START, "model")
builder.add_edge("model", "summary")
builder.add_edge("summary", END)

graph = builder.compile(checkpointer=memory)
```

> Notes - `HumanNode` declares the schema of what the **user** must
> provide. - The frontend receives a **node event** when execution
> reaches this node. - Use persistence so the run can pause indefinitely
> until the user responds.

### JavaScript/TypeScript (LangGraph)

``` ts
import { StateGraph, START, END, HumanNode } from "@langgraph/sdk";

interface S {
  input: string;
  summary?: string;
}

const model = async (s: S) => ({ input: s.input });

const summaryNode = new HumanNode<S>({
  name: "summary",
  description: "User provides a summary of the input text",
  schema: { summary: "string" },
});

const g = new StateGraph<S>()
  .addNode("model", model)
  .addNode("summary", summaryNode)
  .addEdge(START, "model")
  .addEdge("model", "summary")
  .addEdge("summary", END)
  .compile({ /* checkpointer */ });
```

------------------------------------------------------------------------

## Frontend: resolve a HumanNode

Use the `useLangGraphNode` hook to catch node events and present custom
UI.

``` tsx
import { useLangGraphNode } from "@copilotkit/react";

export function SummarySurface() {
  useLangGraphNode({
    onNode: async (evt, { resolve }) => {
      if (evt.name === "summary") {
        const text = await openSummaryModal(evt.value);
        resolve({ summary: text });
      }
    },
  });
  return null;
}
```

> Tips - The `evt` payload includes `name`, `description`, and expected
> `schema`. - Call `resolve(object)` with keys matching the schema. -
> You can implement arbitrary UI (modal, inline form, chat card, etc.).

------------------------------------------------------------------------

## Example: Chat card renderer

``` tsx
import { useCopilotAction } from "@copilotkit/react";

useCopilotAction({
  name: "get_summary",
  parameters: { input: "string" },
  render: ({ input }, { resolve }) => (
    <div className="rounded-2xl p-4 shadow">
      <h3 className="font-semibold mb-2">Summarize this text:</h3>
      <p className="mb-4">{input}</p>
      <textarea id="summary" className="w-full border p-2" />
      <button
        onClick={() =>
          resolve({ summary: (document.getElementById("summary") as HTMLTextAreaElement).value })
        }
        className="mt-2 px-3 py-1 rounded bg-blue-500 text-white"
      >
        Submit
      </button>
    </div>
  ),
});
```

------------------------------------------------------------------------

## Common patterns

-   **Manual override**: let users correct or enrich model outputs.
-   **Data entry tasks**: structured forms (dates, names, amounts).
-   **Approval workflows**: model drafts → user fills gaps → graph
    continues.
-   **Progressive UIs**: multi‑step flows, each step as a HumanNode.

------------------------------------------------------------------------

## Persistence & threading

-   Always enable checkpointing so runs can pause for long periods.
-   Ensure frontend and backend share the same **thread / run id**.
-   Store partial state in append‑only fashion for reliability.

------------------------------------------------------------------------

## Troubleshooting

-   **Graph hangs at node** → UI must call `resolve(...)` with matching
    schema.
-   **Wrong shape in value** → Ensure keys in object exactly match
    schema fields.
-   **UI not showing** → Verify your hook listens for the correct node
    `name`.
-   **Resume lost after reload** → Reconnect with the same thread id.

------------------------------------------------------------------------

## Minimal E2E checklist

-   [ ] Backend defines `HumanNode(schema)` in graph
-   [ ] Persistence is configured (checkpoint / thread id)
-   [ ] Frontend listens with `useLangGraphNode`
-   [ ] UI calls `resolve(object)` with schema‑compliant value
-   [ ] Graph resumes and consumes user‑provided state

------------------------------------------------------------------------

## Further reading

-   LangGraph --- HumanNode API reference & HITL how‑tos
-   CopilotKit --- `useLangGraphNode` hook & Generative UI patterns

------------------------------------------------------------------------

### License & attribution

This doc is a derivative summary intended for internal use, based on the
CopilotKit + LangGraph node flow documentation and related references.
