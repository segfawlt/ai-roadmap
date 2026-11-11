# Assignment: Harness Engineering

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Agent Harness for Reliable Autonomous Work |
| **Course** | AI Advanced |
| **Project Name** | `harness-engineering-lab` |
| **Estimated Time** | 120 minutes |
| **Framework** | Python 3.11+, LangChain 1.x, Anthropic (or OpenAI) API |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Design** a specification file that guides agent behavior

- **Implement** context management with bounded history and condensation

- **Build** safety constraints including tool validation and human-in-the-loop gates

- **Create** a trace logger for agent observability

- **Analyze** agent trajectories to identify failure modes

------------------------------------------------------------------------

## Problem Description

You are building a harness for a coding assistant agent that can read files, write files, and run shell commands. The agent must operate within safety boundaries, manage its context window as it works through a multi-step task, and produce observable traces for post-hoc analysis. Your job is to engineer the environment — not the model.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain[anthropic]` \>= 1.0 \# or `langchain[openai]`

  - `tiktoken` \>= 0.7.0

### Test Project

Create a small test project (3–5 Python files) that the agent will work on. The project should have at least one bug for the agent to fix.

------------------------------------------------------------------------

## Tasks

### Task 1: Specification File (20 points)

1.  **Write a `CLAUDE.md`** for the test project containing:

    - Build and test commands

    - File naming conventions and code style rules

    - Constraints (what the agent must not do)

    - At least 2 examples of desired behavior

2.  **Write a tool manifest** that defines:

    - Which tools the agent can use (read_file, write_file, run_command)

    - Parameter constraints per tool (allowed directories, blocked commands)

    - Which actions require human approval

3.  **Test** that the spec is clear enough for a colleague to follow manually

### Task 2: Context and Memory Management (25 points)

1.  **Implement bounded conversation history** that:

    - Tracks token usage per message

    - Triggers condensation when history exceeds 80% of budget

    - Always preserves system message and last 3 exchanges

    - Summarizes older messages rather than dropping them

2.  **Implement a scratchpad** that:

    - Persists key decisions and findings to a JSON file

    - Can be loaded into context on resumption

    - Tracks completed subtasks

3.  **Demonstrate** the history manager working across a 15-turn conversation without context overflow

### Task 3: Constraints and Safety (25 points)

1.  **Implement tool validation** that:

    - Checks every tool call against the manifest from Task 1

    - Blocks calls to unauthorized tools

    - Blocks dangerous parameters (e.g., `rm`` ``-rf`, `git`` ``push`` ``--force`)

    - Logs all blocked actions with reasons

2.  **Implement human-in-the-loop gates** for:

    - File deletion

    - Any command that modifies git state

    - Writing to files outside the project directory

3.  **Test with adversarial inputs**: craft 5 tool calls that should be blocked and verify the harness catches all of them

### Task 4: Observability and Trajectory Analysis (30 points)

1.  **Build a trace logger** that records for each agent step:

    - Timestamp, action name, input parameters, output

    - Token usage and latency

    - Whether the action was approved, blocked, or errored

2.  **Run the agent** on a bug-fix task (at least 8 steps) and save the trace

3.  **Analyze the trace** and produce a report with:

    - Total steps, tokens, and duration

    - Tool error rate and blocked action count

    - Context utilization over time (tokens used / budget)

    - Identification of wasted steps (actions that didn’t contribute to the goal)

    - Recommendations for harness improvements

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

`CLAUDE.md` specification file

Tool manifest (JSON or Python dict)

Source code for context manager, safety layer, and trace logger

Agent trace file (JSON)

Trajectory analysis report

`README.md` with setup and usage instructions

### Submission Checklist

All code runs without errors

Spec file is complete and testable

Context manager handles overflow gracefully

Safety layer blocks all 5 adversarial inputs

Trace logger captures all agent steps

Analysis report includes actionable recommendations

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                              | Points  |
|---------------------------------------|---------|
| Specification file and tool manifest  | 20      |
| Context and memory management         | 25      |
| Constraints and safety implementation | 25      |
| Observability and trajectory analysis | 30      |
| **Total**                             | **100** |

------------------------------------------------------------------------

## Hints

> **Tip**
>
> Tip
>
> - Start with the spec file — it defines everything else
>
> - For context management, use `tiktoken` to count tokens accurately
>
> - For the safety layer, think about what a malicious prompt injection might try to do
>
> - The trace logger should be a decorator or middleware, not mixed into business logic
>
> - When analyzing the trace, look for loops where the agent retries the same failing action
