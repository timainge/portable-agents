# Portable Agents: Vision & Architecture Roadmap

## 1. Core Philosophy: The "Agent Compiler"
We are building an **"Agent IR (Intermediate Representation) → Runtime Backend" compiler**.
Instead of locking into a single framework (LangGraph, OpenAI, etc.), we define agents in a neutral, declarative **AgentSpec**. We then build **Adapters** that "compile" or "interpret" this spec for specific runtimes.

### Key Architectural Decisions

#### 1.1 Text-Based Orchestration > Rigid Graphs
**The Insight:** Most frameworks over-engineer control flow with complex graphs (nodes, edges, conditional jumps). This is hard to port.
**Our Approach:** We treat orchestration as **data** in a text-based **Scratchpad**.
*   **State is Text:** The agent maintains a `Scratchpad` (Goal, Plan, Todo, Status) in its context.
*   **Logic is Prompting:** The "orchestrator" is just a loop that asks the model: "Given this scratchpad, what is the next move?"
*   **Portability:** Every LLM and framework can handle "read text, decide action, update text". We don't need to port graph execution logic; we just port the prompt and the loop.

#### 1.2 Memory as Tools
**The Insight:** Frameworks implement memory differently (RAG, vector stores, sessions).
**Our Approach:** We abstract memory as **standardized tools**.
*   Instead of a "Memory Module", the agent sees tools: `search_knowledge()`, `save_fact()`, `recall_profile()`.
*   **The Adapter's Job:** The adapter maps these tool calls to the specific backend (e.g., OpenAI Vector Store, Azure AI Search, or a local ChromaDB).
*   **Benefit:** The agent's cognitive architecture remains identical regardless of the storage backend.

#### 1.3 Usage-Driven Specs
**The Insight:** Don't try to model every feature of every framework.
**Our Approach:** Define the spec based on *what the agent needs to do*.
*   Start with the capabilities we need (e.g., "Plan a task", "Search docs").
*   Define the neutral spec for those.
*   Map them down to frameworks.
*   Accept that some "proprietary" features (e.g., specific Azure security hooks) might remain outside the portable core.

## 2. The "Tricky Middle" Analysis
We identified three tiers of portability:

*   **Tier 1: The Easy Core (Highly Portable)**
    *   Model calls with prompts.
    *   Structured tool calls (JSON Schema).
    *   Short-term chat history/sessions.
    *   *Strategy:* Map these 1:1 in adapters.

*   **Tier 2: The Tricky Middle (Requires Adapters)**
    *   **Memory Services:** Requires mapping abstract tools to concrete DBs/APIs.
    *   **Observability:** Tracing/Logging schemas vary wildy. We need a unified event schema.
    *   **Orchestration:** We bypass framework-native planners (like AutoGen's conversation patterns) in favor of our Scratchpad, but we still need to hook into their event loops.

*   **Tier 3: The Proprietary Layer (Non-Portable)**
    *   Direct integrations (e.g., Microsoft 365 Graph).
    *   Cloud-specific security/governance.
    *   *Strategy:* Mark these as "platform-specific extensions" in the spec.

## 3. Long-Term Roadmap

### Phase 1: The POC (Current Focus)
*   **Goal:** Prove the "Spec + Adapter" model works for two distinct runtimes.
*   **Target:** PydanticAI (Local) + OpenAI Agents SDK (Hosted).
*   **Deliverable:** An Architect Agent that runs on both and behaves identically.

### Phase 2: The Meta-Agent Strategy
*   **Goal:** Scale the spec without manual toil.
*   **Concept:** Build a "Framework Profiler" agent.
    *   It reads documentation of new frameworks (Anthropic, Google ADK, etc.).
    *   It maps their features to our AgentSpec.
    *   It generates the draft Adapter code.
*   **Benefit:** We use our own stack to maintain our stack.

### Phase 3: The "MCP-Native" Future
*   **Goal:** Align with the Model Context Protocol (MCP).
*   **Strategy:** Use MCP as the "wire format" for our tools.
    *   If OpenAI/Anthropic adopt MCP natively, our "Adapters" become trivial configuration.
    *   Our spec becomes a recipe for generating MCP servers.

## 4. "Vibes" & Design Ethos
*   **Pragmatism over Purity:** If a framework feature is super useful but not portable, expose it as an extension, don't ban it.
*   **Text is the Universal Interface:** When in doubt, solve it with a prompt and a schema, not a class hierarchy.
*   **Agent-First Development:** We should use the Architect Agent to build the system. If the agent can't understand our spec, the spec is too complex.
