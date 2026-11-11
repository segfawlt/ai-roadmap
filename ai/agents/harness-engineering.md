# Harness Engineering

Harness engineering is the practice of shaping the environment around AI agents so they can work reliably. It sits at the intersection of context engineering, evaluation, observability, orchestration, safe autonomy, and software architecture.

The key insight: performance gaps in agent systems are often **harness problems rather than model problems**. The infrastructure choices — how you manage context, constrain tools, persist state, and observe behavior — matter as much as the model itself.

## Learning Objectives

- Define harness engineering and its relationship to context engineering

- Design context and memory management strategies for long-running agents

- Implement constraints and safety boundaries for autonomous agent work

- Write specification files (AGENTS.md, CLAUDE.md) that guide agent behavior

- Build evaluation and observability stacks for multi-step agent trajectories

- Understand runtime infrastructure for durable, resumable agent execution

## 1. What Is a Harness?

A harness is everything surrounding the model that shapes its behavior: the system prompt, available tools, safety constraints, state management, evaluation hooks, and orchestration logic. The model is the engine; the harness is the chassis, steering, and brakes.

``` mermaid
        graph TB
    subgraph Harness["Harness (you engineer this)"]
        CTX["Context & Memory"]
        CON["Constraints & Safety"]
        SPEC["Specifications"]
        EVAL["Evaluation & Observability"]
        RT["Runtime & Orchestration"]
    end
    MODEL["LLM / Agent"] --- Harness
    Harness --> OUTPUT["Reliable Output"]
```

### Why Harness Engineering Matters

| Without Harness | With Harness |
|----|----|
| Agent drifts off-task after 20 turns | Context condensation keeps the agent focused |
| Agent executes destructive commands | Sandbox and tool boundaries prevent harm |
| No visibility into multi-step reasoning | Trace logging and step-level scoring |
| Agent hallucinates tool usage | Spec files define available tools and conventions |
| Failures lose all progress | Checkpointing and durable execution enable resumption |

## 2. Context and Memory Management

Agents use the context window as working memory. In long-running tasks (coding sessions, research, multi-file refactors), the context fills up and the agent loses track of earlier decisions. Harness engineering treats context as a **budget to be managed**, not a buffer to be filled.

### Bounded Conversation Design

Limit how much history the agent carries:

``` python
from langchain.messages import SystemMessage, HumanMessage, AIMessage


def bounded_history(
    messages: list,
    max_tokens: int = 80_000,
    token_counter=None,
) -> list:
    """Keep conversation within budget by summarizing old turns."""
    total = sum(token_counter(m.content) for m in messages)

    if total <= max_tokens:
        return messages

    # Always keep system message and last N turns
    system = [m for m in messages if isinstance(m, SystemMessage)]
    recent = messages[-6:]  # last 3 exchanges
    old = [m for m in messages if m not in system and m not in recent]

    # Summarize old messages
    summary = summarize_messages(old)
    return system + [HumanMessage(content=f"[Earlier context summary]\n{summary}")] + recent
```

### Context Condensation

When the context window fills, compress rather than truncate:

1.  **Summarize completed subtasks** — replace detailed steps with a one-line result

2.  **Drop tool output bodies** — keep the conclusion, discard raw JSON

3.  **Collapse file contents** — replace full file reads with “read file X, found Y”

4.  **Preserve decisions** — never compress architectural choices or user requirements

### Scratchpads and External Memory

For tasks that exceed any context window, offload state to files:

``` python
import json
from pathlib import Path

SCRATCHPAD = Path(".agent/scratchpad.json")


def save_progress(task_id: str, state: dict) -> None:
    """Persist agent progress to disk."""
    data = json.loads(SCRATCHPAD.read_text()) if SCRATCHPAD.exists() else {}
    data[task_id] = state
    SCRATCHPAD.parent.mkdir(parents=True, exist_ok=True)
    SCRATCHPAD.write_text(json.dumps(data, indent=2))


def load_progress(task_id: str) -> dict | None:
    """Resume from saved state."""
    if not SCRATCHPAD.exists():
        return None
    data = json.loads(SCRATCHPAD.read_text())
    return data.get(task_id)
```

## 3. Constraints and Safety

Autonomous agents need boundaries. A harness defines what the agent **can** and **cannot** do, preventing catastrophic actions while allowing productive work.

### Tool Boundaries

Restrict which tools are available and what parameters they accept:

``` python
ALLOWED_TOOLS = {
    "read_file": {"max_size_kb": 500},
    "write_file": {"allowed_dirs": ["src/", "tests/"]},
    "run_command": {"blocked": ["rm -rf", "git push --force", "DROP TABLE"]},
    "web_search": {"max_results": 10},
}


def validate_tool_call(tool_name: str, params: dict) -> bool:
    """Check if a tool call is within allowed boundaries."""
    if tool_name not in ALLOWED_TOOLS:
        return False

    constraints = ALLOWED_TOOLS[tool_name]

    if "blocked" in constraints:
        command = params.get("command", "")
        if any(blocked in command for blocked in constraints["blocked"]):
            return False

    if "allowed_dirs" in constraints:
        path = params.get("path", "")
        if not any(path.startswith(d) for d in constraints["allowed_dirs"]):
            return False

    return True
```

### Sandboxing

Run agent actions in isolated environments:

| Strategy | Use Case | Trade-off |
|----|----|----|
| Docker containers | Code execution, file system changes | Heavier setup, full isolation |
| Git worktrees | Code changes with easy rollback | Lightweight, git-native |
| Temporary directories | File operations | Simple, no persistence |
| VM snapshots | System-level operations | Expensive, maximum isolation |

### Human-in-the-Loop Gates

Not everything should be autonomous. Define escalation points:

``` python
REQUIRES_APPROVAL = [
    "delete_file",
    "modify_ci_config",
    "push_to_remote",
    "run_migration",
    "send_message",
]


async def execute_with_gates(tool_name: str, params: dict, agent_context) -> dict:
    """Execute tool call, requiring human approval for sensitive actions."""
    if tool_name in REQUIRES_APPROVAL:
        approved = await agent_context.request_approval(
            action=tool_name,
            params=params,
            reason=f"Agent wants to {tool_name} with {params}",
        )
        if not approved:
            return {"status": "blocked", "reason": "User denied action"}

    return await agent_context.execute_tool(tool_name, params)
```

## 4. Specifications and Workflow Design

Spec files are the **written contract** between the human and the agent. They define conventions, available tools, project structure, and decision-making rules.

### Repo-Local Instruction Files

Modern agent systems read instruction files from the repository:

| File | Purpose | Example Content |
|----|----|----|
| `CLAUDE.md` | Claude Code project instructions | Build commands, file conventions, commit style |
| `AGENTS.md` | Multi-agent coordination rules | Agent roles, handoff protocols, shared state |
| `.cursorrules` | Cursor IDE agent instructions | Coding style, framework preferences |
| `copilot-instructions.md` | GitHub Copilot workspace config | Language preferences, test patterns |

### Effective Spec Design

A good spec file:

1.  **Defines the environment** — build commands, test commands, deploy process

2.  **States conventions** — file naming, commit format, code style

3.  **Lists constraints** — what not to do, what requires approval

4.  **Provides examples** — show the agent what “good” looks like

``` markdown
# CLAUDE.md — Example Structure

## Build Commands

- `npm run build` — production build
- `npm test` — run test suite

## Conventions

- TypeScript strict mode, no `any`
- Tests co-located with source: `foo.ts` → `foo.test.ts`
- Conventional commits: `feat:`, `fix:`, `docs:`

## Constraints

- Never modify `package-lock.json` manually
- Never push to main directly
- Ask before deleting files
```

### Spec-Driven Development

The workflow: write the spec first, then let the agent execute within its boundaries.

``` mermaid
        graph LR
    SPEC["Write Spec<br/>(human)"] --> PLAN["Agent Plans<br/>within constraints"]
    PLAN --> EXEC["Agent Executes<br/>with tool boundaries"]
    EXEC --> REVIEW["Human Reviews<br/>at gates"]
    REVIEW -->|Approve| MERGE["Merge"]
    REVIEW -->|Reject| PLAN
```

## 5. Evaluation and Observability

Evaluating agents is harder than evaluating single LLM calls because agents take **multi-step trajectories** where each step depends on previous ones.

### Trajectory-Level Evaluation

Score the entire sequence of actions, not just the final output:

``` python
from dataclasses import dataclass


@dataclass
class AgentStep:
    action: str
    input: dict
    output: dict
    duration_ms: int
    tokens_used: int


@dataclass
class AgentTrajectory:
    steps: list[AgentStep]
    final_output: str
    total_duration_ms: int
    total_tokens: int


def score_trajectory(trajectory: AgentTrajectory, rubric: dict) -> dict:
    """Score an agent trajectory across multiple dimensions."""
    scores = {}

    # Task completion: did the agent achieve the goal?
    scores["completion"] = rubric["completion_check"](trajectory.final_output)

    # Efficiency: how many steps did it take?
    scores["efficiency"] = min(1.0, rubric["optimal_steps"] / len(trajectory.steps))

    # Safety: did any step violate constraints?
    violations = [s for s in trajectory.steps if is_violation(s, rubric["constraints"])]
    scores["safety"] = 1.0 if not violations else 0.0

    # Cost: total token usage
    scores["cost"] = min(1.0, rubric["token_budget"] / trajectory.total_tokens)

    return scores
```

### Trace Logging

Log every agent action for post-hoc analysis:

``` python
import json
import time
from pathlib import Path


class TraceLogger:
    """Log agent actions for observability."""

    def __init__(self, trace_dir: Path):
        self.trace_dir = trace_dir
        self.trace_dir.mkdir(parents=True, exist_ok=True)
        self.session_id = f"trace_{int(time.time())}"
        self.steps = []

    def log_step(self, action: str, input_data: dict, output_data: dict) -> None:
        step = {
            "timestamp": time.time(),
            "action": action,
            "input": input_data,
            "output": output_data,
            "step_number": len(self.steps) + 1,
        }
        self.steps.append(step)

    def save(self) -> Path:
        path = self.trace_dir / f"{self.session_id}.json"
        path.write_text(json.dumps(self.steps, indent=2))
        return path
```

### What to Measure

| Metric                  | What It Tells You                           |
|-------------------------|---------------------------------------------|
| Task completion rate    | Does the agent finish the job?              |
| Steps per task          | Is the agent efficient or wandering?        |
| Tool error rate         | Are tool boundaries too tight or too loose? |
| Context utilization     | Is the agent running out of context?        |
| Human intervention rate | How often does the agent need help?         |
| Cost per task           | Is the agent economically viable?           |
| Safety violation rate   | Are constraints holding?                    |

## 6. Benchmarks

Standard benchmarks measure agent capabilities across domains:

| Benchmark | Domain   | What It Tests                         |
|-----------|----------|---------------------------------------|
| SWE-bench | Coding   | Resolve real GitHub issues end-to-end |
| WebArena  | Web      | Complete tasks on live websites       |
| OSWorld   | Desktop  | Interact with desktop applications    |
| GAIA      | General  | Multi-step reasoning with tool use    |
| Cybench   | Security | Capture-the-flag security challenges  |
| HumanEval | Coding   | Function-level code generation        |

Benchmark results depend heavily on the harness. The same model scores differently with different tool sets, context strategies, and retry policies. **When comparing agents, you are comparing harnesses as much as models.**

## 7. Runtimes and Orchestration

Production agents need infrastructure for durability, state management, and multi-agent coordination.

### Durable Execution

Long-running agents must survive interruptions:

``` python
import json
from pathlib import Path


class CheckpointManager:
    """Save and restore agent state for durable execution."""

    def __init__(self, checkpoint_dir: Path):
        self.checkpoint_dir = checkpoint_dir
        self.checkpoint_dir.mkdir(parents=True, exist_ok=True)

    def save(self, session_id: str, state: dict) -> None:
        path = self.checkpoint_dir / f"{session_id}.json"
        path.write_text(json.dumps(state, indent=2))

    def restore(self, session_id: str) -> dict | None:
        path = self.checkpoint_dir / f"{session_id}.json"
        if path.exists():
            return json.loads(path.read_text())
        return None

    def resume_or_start(self, session_id: str, initial_state: dict) -> dict:
        """Resume from checkpoint or start fresh."""
        saved = self.restore(session_id)
        if saved:
            print(f"Resuming from checkpoint: {len(saved.get('completed_steps', []))} steps done")
            return saved
        return initial_state
```

### Multi-Agent Coordination

When multiple agents collaborate, the harness manages handoffs and shared state:

``` mermaid
        graph TB
    ORCH["Orchestrator<br/>(routes tasks)"]
    ORCH --> A1["Planner Agent<br/>Decomposes task"]
    ORCH --> A2["Coder Agent<br/>Writes implementation"]
    ORCH --> A3["Reviewer Agent<br/>Checks quality"]
    A1 -->|"plan.md"| SHARED["Shared State<br/>(files, DB, message queue)"]
    A2 -->|"code changes"| SHARED
    A3 -->|"review feedback"| SHARED
    SHARED --> ORCH
```

### Agent Loop Patterns

| Pattern | Description | When to Use |
|----|----|----|
| **ReAct loop** | Think → Act → Observe → repeat | Single-agent, tool-use tasks |
| **Plan-and-execute** | Plan all steps → execute sequentially | Well-defined, decomposable tasks |
| **Reflexion** | Execute → self-critique → retry | Tasks requiring quality iteration |
| **Supervisor** | One agent delegates to specialized sub-agents | Complex multi-domain tasks |

## 8. Putting It All Together

A production harness integrates all these layers:

``` mermaid
        graph TB
    A["Specifications<br/>(CLAUDE.md, AGENTS.md)<br/><em>What the agent knows</em>"]
    B["Context &amp; Memory<br/>(budgets, condensation)<br/><em>What the agent remembers</em>"]
    C["Constraints &amp; Safety<br/>(sandbox, gates, tools)<br/><em>What the agent can do</em>"]
    D["Runtime<br/>(checkpoints, orchestration, loops)<br/><em>How the agent executes</em>"]
    E["Evaluation &amp; Observability<br/>(traces, scores)<br/><em>How you know it's working</em>"]
    F(["LLM / Agent"])

    A --> B --> C --> D --> E --> F

    style A fill:#1a2a3a,stroke:#42a5f5,color:#e0e0e0
    style B fill:#1a2a3a,stroke:#42a5f5,color:#e0e0e0
    style C fill:#1a2a3a,stroke:#42a5f5,color:#e0e0e0
    style D fill:#1a2a3a,stroke:#42a5f5,color:#e0e0e0
    style E fill:#1a2a3a,stroke:#42a5f5,color:#e0e0e0
    style F fill:#2a1a3a,stroke:#ab47bc,color:#e0e0e0
```

The harness is not a one-time setup — it evolves with the agent. As you discover failure modes, you tighten constraints. As you build trust, you relax gates. The goal is not perfect control, but **reliable autonomy**.

## Practice

1.  Write a `CLAUDE.md` spec file for a project you work on. Include build commands, conventions, constraints, and examples. Test it by running Claude Code against the project and observing whether the agent follows the spec.

2.  Implement a checkpoint manager that saves agent state after each tool call and can resume from the last checkpoint. Test it by interrupting an agent mid-task and verifying it resumes correctly.

3.  Build a trace logger that records every agent action with timestamps. After a 10-step agent session, analyze the trace to identify: wasted steps, context utilization, and tool error rate.
