# PRD: Cross-Framework Agent Spec & Architect Agent POC

## 1. Executive Summary
We aim to create a **neutral, cross-framework agent specification** that allows defining AI agents once and deploying them to multiple runtimes. To validate this, we will build an **Architect/Research Agent** POC that runs on both **PydanticAI** (local/OSS) and **OpenAI Agents SDK** (hosted).

The core innovation is moving away from rigid, framework-specific graphs to **text-based orchestration** (planning scratchpads) and treating memory/knowledge as **standardized tools**.

> **Note:** For the long-term vision, architectural philosophy, and "meta-agent" strategy, please refer to [agent_vision_and_roadmap.md](agent_vision_and_roadmap.md). This PRD focuses strictly on the execution of the Sprint 1 POC.

## 2. Goals & Non-Goals

### Goals (Sprint 1)
1.  **Define AgentSpec v0**: A neutral YAML/JSON schema for agents, covering persona, prompts, tools, memory, and planning.
2.  **Implement Architect Agent POC**: An agent that can research frameworks and evolve the spec, running on:
    *   **PydanticAI**: Proving local, type-safe Python execution.
    *   **OpenAI Agents SDK**: Proving hosted, session-based execution.
3.  **Demonstrate Portability**: Show that the *same* spec drives both implementations with minimal adapter logic.
4.  **Validate "Text-as-Orchestration"**: Prove that a structured scratchpad (Goal/Plan/Todo/Status) is a viable, portable alternative to complex graphs.

### Non-Goals (Sprint 1)
*   Supporting every framework immediately (start with just two).
*   Complex multi-agent graphs (focus on single powerful agent first).
*   Production-grade infrastructure (in-memory/local storage is fine for POC).

## 3. Core Concepts

### 3.1 The Neutral Agent Spec (AgentSpec v0)
A declarative definition of an agent.
*   **Identity**: Name, model config (e.g., `gpt-4o`), persona/system prompt.
*   **Prompts**: Named templates (e.g., `plan`, `act`, `reflect`).
*   **Tools**: JSON-Schema definitions (compatible with MCP).
*   **Memory**: Capabilities defined as tools (e.g., `search_knowledge`, `index_fact`).
*   **Orchestration**: A text-based protocol (Scratchpad) rather than a code graph.

### 3.2 Text-Based Orchestration
Instead of framework-specific state machines, the agent maintains a **Scratchpad** in its context:
```json
{
  "goal": "Research PydanticAI features",
  "plan": ["Read docs", "Compare with OpenAI SDK", "Update Spec"],
  "current_step": "Read docs",
  "status": "IN_PROGRESS",
  "memory_context": "..."
}
```
The "Orchestrator" is simply a prompt loop that reads this state, decides the next action (tool call or update state), and iterates. This is universally portable.

### 3.3 Memory as Tools
Memory is not a framework feature but a set of tools:
*   `search_internal_knowledge(query)`
*   `save_to_memory(content, tags)`
*   `search_web(query)`

Adapters map these to the underlying runtime (e.g., OpenAI Vector Stores, generic pgvector, or an MCP server).

## 4. Technical Architecture

### 4.1 AgentSpec Schema (Draft)
```yaml
agent:
  id: "architect_agent"
  model: "gpt-4o"
  system_prompt: "You are an expert AI architect..."
  
  orchestration:
    type: "text_scratchpad"
    schema_ref: "planning_schema_v1"

  tools:
    - id: "search_docs"
      description: "Search framework documentation"
      input_schema: { ... }
    - id: "manage_spec"
      description: "Read/Update the AgentSpec file"
      input_schema: { ... }

  memory:
    - id: "project_knowledge"
      type: "vector_store"
      access: ["read", "write"]
```

### 4.2 Adapters
*   **PydanticAI Adapter**:
    *   Maps `AgentSpec` -> `pydantic_ai.Agent`.
    *   Tools -> Python functions with `pydantic.BaseModel` inputs.
    *   Loop -> Custom Python `while` loop managing the scratchpad.
*   **OpenAI Agents SDK Adapter**:
    *   Maps `AgentSpec` -> `client.agents.create()`.
    *   Tools -> OpenAI Function definitions.
    *   Loop -> OpenAI `run` loop (server-side orchestration).

## 5. Sprint 1: Execution Details

### 5.1 Proposed File Structure
```
/
├── specs/
│   └── architect_agent.yaml      # The neutral spec
├── src/
│   ├── adapters/
│   │   ├── pydantic_ai/          # PydanticAI implementation
│   │   │   ├── runner.py
│   │   │   └── tools.py
│   │   └── openai_sdk/           # OpenAI SDK implementation
│   │       ├── runner.py
│   │       └── tools.py
│   ├── core/
│   │   ├── spec_loader.py        # YAML -> Python objects
│   │   └── scratchpad.py         # Pydantic model for the scratchpad
│   └── tools/                    # Shared tool implementations (logic only)
│       ├── file_ops.py
│       └── web_search.py
├── main.py                       # CLI entrypoint to run either adapter
└── requirements.txt
```

### 5.2 Dependencies
*   `pydantic`
*   `pydantic-ai`
*   `openai`
*   `pyyaml`
*   `python-dotenv`

### 5.3 Success Criteria for Sprint 1
1.  `python main.py --runtime pydantic` runs the Architect Agent, which successfully reads its own spec and prints a plan.
2.  `python main.py --runtime openai` does the same using the OpenAI SDK.
3.  Both runtimes use the *exact same* `architect_agent.yaml` file.

## 6. POC Implementation Plan

### Phase 1: The Spec & Scratchpad
1.  Define the `AgentSpec` YAML format.
2.  Define the `Scratchpad` Pydantic model (Goal, Plan, Steps, Results).

### Phase 2: PydanticAI Implementation
1.  Create `pydantic_runner.py`.
2.  Implement the "Loop": A function that calls the model, parses the scratchpad update, executes tools, and repeats.
3.  Implement Tools: `search_web` (mock or real), `read_file`, `write_file`.

### Phase 3: OpenAI Agents SDK Implementation
1.  Create `openai_runner.py`.
2.  Map the spec to OpenAI's `Agent` and `Tool` objects.
3.  Use OpenAI's state management but enforce the Scratchpad pattern via system prompt.

### Phase 4: The Architect Agent
1.  Write the "Architect" persona.
2.  Give it the task: "Research PydanticAI and OpenAI Agents SDK and refine this spec."
3.  Run it on both runtimes and compare results.

## 6. Technical Notes & Learnings (from Chat)
*   **Portability Sweet Spot**: Model calls, structured tools, and chat history are highly portable.
*   **The "Tricky Middle"**: Memory services and Observability require adapters.
*   **MCP**: The Model Context Protocol is a strong candidate for the "wire format" of tools, making them portable across Claude, OpenAI, and local tools.
*   **Meta-Agents**: Using an agent to read docs and generate the spec/adapters is a viable bootstrapping strategy.

## 7. Next Steps
1.  Create the `agent_spec.yaml` (v0).
2.  Set up the Python environment for PydanticAI and OpenAI SDK.
3.  Build the PydanticAI runner first (easiest to debug).
