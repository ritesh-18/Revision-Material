# Chapter 11 — AI Agents

## 11.1 Concept Explanation

An AI agent is an LLM that observes, decides, and acts in a loop, using tools to interact with the world. The model is no longer a one-shot completion engine; it is a planner that can call functions, read their results, and iterate. Agents are how GenAI moves from answering questions to *doing things* — booking a flight, refactoring a codebase, processing an invoice, debugging a system.

The hard part of agents is not the loop. It is everything around it: reliability, observability, cost, security, debuggability, and the fact that small reasoning errors compound across many steps.

## 11.2 The Anatomy of an Agent

```
   User goal
        |
        v
   +-----------+
   |  Planner  |  <-- decides next action
   +-----------+
        |
        v
   +-----------+
   |   Tools   |  <-- search, code, calls, etc.
   +-----------+
        |
        v
   +-----------+
   |  Memory   |  <-- short-term context, long-term notes
   +-----------+
        |
        v
   Outcome / next loop
```

Every agent system has these four parts: planner, tools, memory, and a loop. Frameworks differ in how opinionated they are about each.

## 11.3 Tool Calling

Tool calling (also known as function calling) is the mechanism by which an LLM emits a structured request for an external action instead of plain text. The runtime parses this request, executes the corresponding tool, and feeds the result back into the model's context.

Modern frontier models support tool calling natively. The developer defines tools as a schema (name, description, parameters). The model decides when to call which tool, with what arguments. Tools can be anything: HTTP calls, database queries, code execution sandboxes, file operations, even calls to other LLMs.

A good tool definition is small, well-named, and well-described. Vague descriptions produce wrong tool choices. Too many tools in one prompt produce confusion. Best practice: keep tool count under 20 in a single agent; if you need more, partition into specialized sub-agents.

## 11.4 Planning Systems

A planner decides the sequence of actions. Three planning approaches:

**ReAct (Reason + Act).** The model alternates reasoning steps with action steps. At each step, the model thinks aloud, picks an action, observes the result, and continues. Simple and effective for moderate complexity.

**Plan-and-Execute.** The model produces a full plan up front, then executes step by step. Better for known-structure tasks, but rigid when the world surprises you.

**Tree of Thoughts / Graph of Thoughts.** Multiple candidate plans are explored, scored, pruned. Higher quality on hard reasoning, much higher cost.

**Hierarchical agents.** A top-level planner delegates sub-tasks to specialist sub-agents. Useful for complex multi-domain work.

In production, ReAct or Plan-and-Execute dominates because they are simpler to debug and cheaper to run. Fancy planning is research-paper material; ReAct is what ships.

## 11.5 Reflection

Reflection is the agent reviewing its own outputs and revising. After producing a result, the agent (or a separate critic model) asks: "is this correct, complete, and aligned with the goal?" If not, retry with the critique as additional context.

Reflection improves quality, especially on tasks with subjective quality (writing, code review, complex analysis). It also at least doubles latency and cost, since you generate twice. Use selectively.

## 11.6 Memory

Agents need memory at multiple time horizons.

**Working memory.** The current context window. Holds the active task, recent observations, intermediate results. Fits within the model's context limit.

**Short-term memory.** Recent conversation history beyond the immediate task. Often summarized aggressively to fit.

**Long-term memory.** Persistent facts about the user, past projects, learned preferences. Stored externally, retrieved like RAG.

**Episodic memory.** Specific past events ("last time we tried X, it failed because Y").

**Procedural memory.** How to perform certain tasks well — playbooks the agent has been given or has learned.

A weakness of current models is that long sessions degrade. Memory must be actively compressed, indexed, and selectively retrieved, not just appended.

## 11.7 Multi-Agent Systems

A multi-agent system has multiple LLM-powered agents collaborating. Patterns:

**Sequential pipeline.** Agent A's output is Agent B's input. Useful for staged workflows (research, then draft, then critique).

**Hierarchical / orchestrator-worker.** A supervisor agent delegates subtasks to specialist agents and combines results. The dominant pattern for complex products.

**Debate / consensus.** Multiple agents argue or vote, producing a consensus answer. Improves quality on contested questions. Expensive.

**Swarm.** Many simple agents act in parallel with light coordination. Used for parallelizable workloads (web crawling, batch analysis).

Multi-agent systems multiply cost and latency. They are justified when single-agent quality plateaus. Most production agent systems should start single-agent and add agents only when measurement demands it.

## 11.8 Agent Communication

Inside a multi-agent system, agents must exchange messages. Two approaches:

**Shared memory.** All agents read and write a common scratchpad. Simple but contention-prone and hard to debug.

**Message passing.** Agents send structured messages over a bus. Cleaner architecture, supports parallelism, requires schema discipline.

Standardization efforts like the Model Context Protocol (MCP) define common interfaces for agent-to-tool and agent-to-agent communication.

## 11.9 LangChain

LangChain is the most popular agent framework in Python. Strengths: huge ecosystem, integrations with nearly every LLM, vector DB, and tool. Weaknesses: large API surface, frequent breaking changes, abstractions sometimes hide what is happening.

LangChain is great for prototypes and small agents; production teams often graduate to LangGraph or custom orchestration.

## 11.10 LangGraph

LangGraph is the LangChain team's evolution toward explicit graph-based agent orchestration. Agents are nodes; control flow is edges; state is explicit. Conditional branching, parallel execution, checkpointing, and human-in-the-loop are first-class.

LangGraph is the better choice for non-trivial production agents in the LangChain family because it makes control flow legible and debuggable.

## 11.11 CrewAI

CrewAI focuses on role-based multi-agent crews. You define agents by role ("researcher," "writer," "editor"), tasks, and a process (sequential, hierarchical). Less general than LangGraph but ergonomic for content-generation pipelines.

## 11.12 Semantic Kernel

Microsoft's framework, .NET-first with Python and Java support. Strong in enterprise environments tied to Azure, with Skills (plugins) and Planners as core primitives.

## 11.13 AutoGen

Microsoft Research's framework for multi-agent conversation. Agents talk to each other in structured conversations, with one agent often playing a user-proxy role. Strong for code-execution agents.

## 11.14 Model Context Protocol (MCP)

MCP is an open protocol introduced by Anthropic for connecting LLMs to external tools and data sources in a standardized way. Instead of every agent framework defining its own tool format, MCP provides a shared interface — a tool published as an MCP server can be consumed by any MCP-aware client.

The adoption is rapid. Treat MCP as the emerging default for tool integrations, especially when you need cross-product or cross-team interoperability.

## 11.15 Orchestration Internals

What is the agent actually doing on each turn?

1. Build the prompt: system instructions + tool definitions + memory + conversation history + latest input.
2. Call the LLM with tool-calling enabled.
3. If the response is a final answer, return it.
4. If the response is a tool call, parse the call, validate the arguments, execute the tool, capture the result.
5. Append the tool result to the conversation history.
6. Go back to step 1.

The loop terminates when the model produces a final answer, hits a step limit, or fails. Step limits are essential — without them, a confused agent can loop forever, burning budget.

## 11.16 Distributed Agent Systems

At scale, agents do not live in a single process. Tools are distributed services. Memory is a distributed store. Agent state is checkpointed for resilience.

Patterns:
- **Stateless agent loop, external state.** Each iteration is a service call; state lives in Redis or a workflow engine.
- **Workflow engines (Temporal, Restate, Inngest).** Treat the agent loop as a durable workflow. Retries, timeouts, and resumption become free.
- **Event-driven agents.** Tools publish results to an event bus; agents react. Suited for asynchronous, long-running flows.

A production multi-agent system often looks more like a distributed workflow system than a chatbot.

## 11.17 Failure Handling

Agents fail in many ways:
- Wrong tool chosen.
- Bad tool arguments.
- Tool timeout or error.
- Infinite loops.
- Token budget exhaustion.
- Hallucinated tool names.
- Cascading errors from earlier wrong outputs.

Defenses:
- Validate tool arguments before execution.
- Cap steps and total tokens per session.
- Detect repetitive loops (same tool, same args, multiple times).
- Idempotent tools so retries are safe.
- Sandbox destructive tools or require human approval.
- Comprehensive logging so postmortems are tractable.

## 11.18 Security Concerns

Agent security is hard. The same loop that makes agents powerful makes them dangerous.

- **Prompt injection.** A document or web page can hijack the agent. Treat any retrieved or fetched content as untrusted; do not let it overrule the system prompt.
- **Tool abuse.** A misled agent invokes destructive tools. Restrict the tool surface, require confirmation for destructive operations, scope tool permissions to the user's own permissions.
- **Data exfiltration.** Agents with both search and write tools can be tricked into copying private data to public places. Audit tool combinations.
- **Lateral movement.** Agents with access to many systems can be a single point of compromise. Treat the agent's credentials carefully.

The general principle: assume the model can be tricked; design tools and permissions so that even a misled agent cannot cause unrecoverable harm.

## 11.19 Production Architecture

```
   Client
     |
   Gateway (auth, rate limit)
     |
   Agent service
     |
     +--- LLM provider
     +--- Tool services (search, code, internal APIs)
     +--- Memory store (Redis + vector DB)
     +--- Workflow engine (Temporal) for long-running agents
     +--- Observability (LangSmith, Phoenix, OpenTelemetry)
     +--- Eval and safety pipelines
```

Long-running agents (research tasks, code refactors) benefit from durable workflows. Short interactive agents (chat with tools) can stay in-process.

## 11.20 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Bigger model as planner | Higher quality plans | Higher cost per step |
| More tools | More capability | More confusion, worse tool selection |
| Reflection on every step | Higher quality | 2-3x cost and latency |
| Multi-agent | Specialization | Complexity, debugging difficulty |
| Long memory window | More context | Slower, more expensive |

## 11.21 Scaling Challenges

- Cost grows superlinearly with task complexity.
- Latency is the sum of many LLM calls plus tool latencies.
- Per-user throughput is limited by sequential nature of the loop.
- Debugging requires reconstructing many-step traces.

## 11.22 Cost Optimization

- Cheaper model for sub-tasks, expensive model only for planning.
- Cache deterministic tool results.
- Limit step count aggressively.
- Compress memory before each step.
- Run reflection only on outputs that fail quality checks.

## 11.23 Interview Questions

- What is the difference between ReAct and Plan-and-Execute?
- How do you prevent infinite loops in an agent?
- Design a multi-agent system for automated code review.
- Where do agents fail in production?
- How do you secure an agent that can write to a database?

## 11.24 Hands-on Exercises

1. Design an agent that books a meeting: list the tools, the planning approach, the failure modes.
2. Plan an evaluation harness for a multi-step agent.
3. Map the security boundary of an agent with file-read, file-write, web-search, and email-send tools.

## 11.25 Common Mistakes

- Too many tools — the model picks wrong.
- No step limit.
- Treating retrieved content as trusted.
- Logging only the final answer; intermediate steps are lost.
- Skipping evaluation; "demo works once" is not production.

## 11.26 Enterprise Best Practices

Adopt MCP for tool integrations. Use a workflow engine for long-running agents. Make tool permissions scope to the user. Build agent evaluation harnesses before scaling features. Maintain a dashboard of step count, cost per session, and tool error rate per agent. Treat agent prompts and tool definitions as versioned, reviewed code.
