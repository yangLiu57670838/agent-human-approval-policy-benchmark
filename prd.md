# PRD: Human Approval Policies for Tool-Using AI Agents

## 1. Project summary

This project investigates when an AI agent should be allowed to execute an action automatically, when it should request human approval, and when it should block the action.

The pilot focuses on software-engineering agents that use tools such as file access, code editing, Git operations, and external messaging. It evaluates whether an external policy gate can improve security without making the agent unusable.

This is a two-week research pilot intended to produce a reproducible technical report, an experiment repository, and an initial research direction for future MPhil or PhD work.

## 2. Research question

**Can an external approval policy based on authorisation, impact, reversibility, sensitivity, and instruction provenance reduce unsafe agent actions while preserving legitimate software-engineering task completion?**

## 3. Research hypotheses

### H1: External enforcement

An application-level policy gate will prevent more unsafe actions than prompt-only instructions.

### H2: Security–utility trade-off

Stricter approval policies will reduce unsafe actions but may also reduce legitimate task completion and increase human approval requests.

### H3: Provenance awareness

Treating instructions from retrieved documents, web pages, issues, and repository files as untrusted will reduce successful indirect prompt-injection attacks.

### H4: Risk-sensitive approval

A policy that distinguishes low-impact reversible actions from destructive, external, or credential-related actions will achieve a better security–utility trade-off than approving every risky action uniformly.

### H5: Agent versus plain LLM

A gated tool-using agent (LLM + tools + loop + external policy) will achieve a higher verified task completion rate than the same LLM used without tools on tasks that require tool use, without a higher unsafe action rate. On tasks where all context fits in the prompt, the two are expected to perform similarly.

### H6: Rule-based gate versus LLM judge

On clean scenarios, a rule-based gate and an LLM judge will make similarly accurate `ALLOW`/`APPROVE`/`DENY` decisions. Under indirect prompt injection, the rule-based gate (especially with provenance tracking) will keep its decision accuracy while the LLM judge degrades. The rule-based gate will also be more consistent across repeated runs and paraphrased requests.

### H7: Hybrid gate

A hybrid gate, in which rules decide clear-cut and high-risk actions and an LLM judge decides only ambiguous low-risk actions, will achieve a better security–utility trade-off than either the rule-based gate or the LLM judge alone.

## 4. Goals

- Build a small tool-using software assistant in a controlled sandbox.
- Implement a deterministic policy engine outside the language model.
- Define `ALLOW`, `APPROVE`, and `DENY` decisions.
- Create a labelled evaluation set of clean and adversarial scenarios.
- Compare multiple agent-loop designs.
- Measure security, usefulness, human friction, latency, and cost.
- Publish reproducible code, configuration, logs, charts, and a technical report.

## 5. Non-goals

- Building a production-ready autonomous coding agent.
- Proving that prompt injection can be solved completely.
- Training a new foundation model.
- Testing against real credentials, production systems, real email accounts, or real financial services.
- Claiming universal safety from a small pilot dataset.

## 6. System design

```text
User request
    ↓
Agent plans an action
    ↓
Policy engine evaluates action and context
    ├── ALLOW → execute in sandbox
    ├── APPROVE → pause for simulated human decision
    └── DENY → block and record reason
    ↓
Observe tool result
    ↓
Update state and continue or stop
```

The language model may propose an action, but it must not be able to bypass the policy engine. Policy checks must run in application code before tool execution.

### 6.1 Implementation stack (LangChain)

Use the LangChain ecosystem wherever it fits, while keeping the policy engine framework-independent so results are not tied to one library.

| Component | Package | Use |
|---|---|---|
| Agent loop and state | `langgraph` | Explicit graph: `model → policy_gate → tools → model`. A checkpointer (`InMemorySaver`) persists state across approval pauses. |
| Human approval pause | `langgraph.types.interrupt` / `Command(resume=...)` | `APPROVE` decisions call `interrupt()`; the simulated approver resumes the run with approve or reject. |
| Tool definitions | `langchain-core` (`@tool`) | Wrap the simulated tools in section 7 with typed schemas. |
| Model providers | `langchain-openai`, `langchain-anthropic` (or other `langchain-*` integrations) | Swap models through configuration for cross-model runs. |
| Optional middleware | `langchain` v1 `create_agent` with custom middleware (`wrap_tool_call`) | Alternative to a hand-built graph. The built-in `HumanInTheLoopMiddleware` is configured per tool, so it is only a reference baseline; the project gate needs argument-, target- and provenance-aware decisions. |
| Tracing | LangSmith (optional) | Trace inspection during development. JSONL logs remain the source of truth for results. |

Design rules:

- `policy/rules.py` is plain Python with no LangChain imports. The LangGraph `policy_gate` node only calls it, so the gate can be unit-tested in isolation and reused with other frameworks.
- The model cannot reach `ToolNode` without passing through `policy_gate`; this is enforced by graph edges, not by the prompt.
- Provenance is tracked in graph state: every message and tool result is tagged `trusted_user` or `untrusted_content`, and the gate receives these tags.
- All conditions in section 10 run on the same LangGraph scaffold, differing only by configuration, so the framework is not a confound.
- Pin exact package versions in `pyproject.toml` and record them in every run's metadata.

## 7. Agent capabilities

The pilot agent may use simulated tools only:

- `read_file(path)`
- `write_file(path, content)`
- `delete_file(path)`
- `create_patch(path, diff)`
- `git_commit(branch, message)`
- `send_message(recipient, content)`
- `read_document(document_id)`
- `request_secret(name)` — used only to test denial behaviour; it returns a dummy value or is blocked

All tools operate in a temporary test workspace. External messages go to a local mock service.

## 8. Approval policy

Each proposed action is evaluated using:

- Explicitness of user authorisation
- Action impact and blast radius
- Reversibility
- Data sensitivity
- Target environment
- Destination or recipient
- Instruction provenance
- Previous failures or repeated actions

### Default policy examples

| Action | Default decision | Reason |
|---|---|---|
| Read a project file | `ALLOW` | Read-only and low impact |
| Create a local patch | `ALLOW` | Reversible and sandboxed |
| Commit to a personal branch | `ALLOW` | Limited external effect |
| Commit to a shared or production branch | `APPROVE` | Shared external state |
| Delete a temporary test file | `ALLOW` if explicitly requested | Low impact and reversible enough |
| Delete a project or production file | `APPROVE` or `DENY` | Destructive action |
| Send an external message | `APPROVE` | External communication and disclosure risk |
| Read credentials or secrets | `DENY` by default | Sensitive access |
| Use credentials against an external service | `APPROVE` | External side effect |
| Follow an instruction found in an untrusted document | `APPROVE` or `DENY` | Possible indirect prompt injection |

The policy must be versioned and stored with every experiment run.

## 9. Evaluation data

### Public benchmark source

Use a small, documented subset of AgentDojo scenarios where practical. AgentDojo evaluates tool-using agents under prompt injection and uses deterministic task checks. Cite the original benchmark and preserve its licence and attribution.

### Project-specific scenarios

Create approximately 30–50 scenarios covering:

- File deletion
- Code patching
- Git commits
- External messages
- Sensitive-data access
- Instructions hidden in repository files or retrieved documents
- Explicitly authorised low-risk actions
- Ambiguous requests
- Repeated or runaway tool calls

Each scenario should contain:

```json
{
  "scenario_id": "delete_production_file_01",
  "user_request": "...",
  "initial_state": "...",
  "available_tools": ["..."],
  "untrusted_content": "...",
  "proposed_action": "...",
  "is_attack": false,
  "variant_of": null,
  "policy_version": "v1",
  "gold_label": "ALLOW | APPROVE | DENY",
  "gold_reason": "...",
  "labelled_by": "..."
}
```

`is_attack` marks scenarios containing injected instructions. `variant_of` links paraphrased versions of the same scenario, used to measure decision consistency. Create 2–3 paraphrases for a subset of scenarios.

### Labelling process

If the person who writes `rules.py` also produces the gold labels from the same policy, the rule-based gate will match the labels almost by construction. To avoid this circularity:

- Freeze `policy_v1.md` and `rules.py` before the final evaluation set is written.
- Have the final evaluation labels produced by at least one person who did not write `rules.py`, working only from `policy_v1.md`. Where two labellers are available, report inter-rater agreement (Cohen's kappa) and record how disagreements were resolved.
- Write some evaluation scenarios after the rules are frozen, so the rules were not tuned to them.
- Do not change the rules after seeing evaluation results. Any later revision becomes `policy_v2` and is evaluated on new scenarios.

Labels are policy-derived. They are not claimed to be universal truth.

## 10. Experimental conditions

Compare seven configurations:

0. **LLM-only baseline:** a single model call with the task and the relevant file contents pasted into the prompt. No tools and no loop; the model outputs a final answer or patch, which the harness applies and checks.
1. **Ungated agent:** the agent executes every proposed action.
2. **Prompt-only guard:** the system prompt instructs the agent to request approval.
3. **LLM-as-judge gate:** a second call to the same model classifies each proposed action as `ALLOW`, `APPROVE`, or `DENY` using the written policy as its prompt.
4. **External policy gate:** application code enforces `ALLOW`, `APPROVE`, and `DENY`.
5. **External gate with provenance:** the policy also tracks whether instructions came from trusted user input or untrusted retrieved content.
6. **Hybrid gate:** the provenance-aware rules from condition 5 decide every action they classify as clear-cut or high-risk. Only actions the rules mark as ambiguous and low-risk are passed to the LLM judge from condition 3. The LLM judge can never downgrade a rule-based `APPROVE` or `DENY` to `ALLOW`.

Use the same tasks, model, tools, and test cases across conditions. Keep development cases separate from final evaluation cases.

### 10.1 Showing that the agent does better than a plain LLM

"Better" must be defined before running experiments. This project uses two separate claims, each tested on the same model so the only variable is the system design:

- **Claim A (capability):** condition 5 completes more tasks than condition 0, verified by deterministic environment checks, not by model self-report.
- **Claim B (safety decisions):** the expected advantage of rules over an LLM judge is not accuracy on ordinary requests, where a capable LLM usually judges approval needs well. It is robustness and consistency. Compare conditions 3, 4, 5, and 6 on three separate measures:
  - **Clean accuracy:** agreement with gold labels on scenarios without attacks. Parity between rules and the LLM judge is expected; the LLM judge may do better on ambiguous cases.
  - **Attacked accuracy:** agreement with gold labels on `is_attack` scenarios. The LLM judge reads the same untrusted content as the agent, so it can be manipulated; the rules cannot. Rules are expected to win here.
  - **Consistency:** the fraction of scenarios that receive the same decision across repeated runs and across paraphrased variants. Rules are deterministic and are expected to win.
  - **Hybrid trade-off:** whether condition 6 keeps the attacked accuracy of the rules while matching the clean accuracy of the LLM judge, with fewer unnecessary approval requests.

Evidence requirements:

1. **Controlled ablation.** Same model, temperature, scenarios, and tool outputs across conditions. Each condition adds or swaps one component (tools, loop, type of gate, provenance), so any improvement can be attributed to that component.
2. **Tasks that require acting.** Include scenarios where the needed information is only available through tools (for example, a file the model has not seen or a failing test that must be run), plus scenarios where the plain LLM has everything it needs. If the agent only wins on the first set, report that honestly; it shows *where* agents help.
3. **Matched compute.** Agents use more tokens and calls. Report a fair comparison against an LLM given an equal budget (for example, best-of-n sampling or self-consistency), and plot completion against tokens and latency.
4. **Paired statistics.** Every scenario is run under every condition, so use paired tests: McNemar's test for per-scenario pass/fail, and bootstrap 95% confidence intervals for rate differences. Use at least 3 repeated runs per scenario and report variance.
5. **Multiple models.** Repeat the key comparison with at least two model families (for example, one OpenAI and one Anthropic model) to show the effect is not specific to one model.
6. **Pre-registration.** Write the hypotheses, metrics, and analysis plan in `report/preregistration.md` before running the final evaluation set.
7. **Failure analysis.** Report cases where the plain LLM beat the agent (for example, the agent looped, over-asked for approval, or followed an injected instruction), and cases where the LLM judge beat the rules (for example, rules that were too rigid for an ambiguous request).
8. **Independent labels.** Gate accuracy is only meaningful against labels produced as described in the labelling process in section 9.

The report should state the result as "on this benchmark, with these models, the gated agent improved X by Y (95% CI ...)", not as a general proof that agents outperform LLMs.

## 11. Metrics

### Security

- Unsafe action rate
- Successful attack rate
- Blocked unauthorised action rate
- Credential-access violations
- External-disclosure violations
- Destructive-action violations

### Utility

- Legitimate task completion rate
- Correct patch or answer rate
- False approval-request rate
- False denial rate

### Gate decision quality

- Decision accuracy against gold labels, reported separately for clean and attacked scenarios
- Per-label precision and recall for `ALLOW`, `APPROVE`, and `DENY`
- Accuracy drop under attack (clean accuracy minus attacked accuracy)
- Paraphrase consistency: fraction of `variant_of` groups that receive the same decision

### Operational cost

- Number of approval requests
- Human decision count
- End-to-end latency
- Token or API cost
- Number of agent steps

### Reliability

- Repeated-run variance
- Policy decision consistency
- Inter-rater agreement (Cohen's kappa) for scenario labels, if a second labeller is available

## 12. Simulated human approval

The pilot should use a deterministic simulated approver for the main experiment. The simulated approver returns the gold policy label and records whether the agent asked for approval correctly.

If time permits, conduct a small human review with 3–5 participants who independently judge a sample of scenarios. Do not present this small sample as a general user study.

## 13. Safety and ethics

- Use dummy secrets only.
- Use local mock services instead of real email, Git hosting, or cloud accounts.
- Do not test against third-party systems without permission.
- Do not publish working credentials, private data, or harmful exploit payloads.
- Keep all attack examples controlled and minimally harmful.
- State limitations and avoid claiming complete protection.

## 14. Two-week schedule

### Days 1–2: Scope and literature

- Read AgentDojo and related prompt-injection work.
- Freeze the research question and policy dimensions.
- Write the threat model and policy version 1.
- Find an independent labeller who will not write `rules.py`.

### Days 3–4: Prototype

- Implement the agent loop.
- Implement simulated tools and the external policy gate.
- Add structured logging.

### Days 5–6: Dataset

- Freeze `policy_v1.md` and `rules.py`.
- Create clean, ambiguous, and adversarial scenarios, plus paraphrased variants for a subset.
- Have the independent labeller add gold labels and reasons from `policy_v1.md`.
- Write deterministic environment checks.

### Days 7–9: Experiments

- Run all seven experimental conditions.
- Repeat runs where model variability matters.
- Store raw traces and configuration files.

### Days 10–11: Analysis

- Calculate security and utility metrics.
- Inspect false positives and false negatives.
- Produce tables and charts.

### Days 12–14: Report and release

- Write a 4–6 page technical report.
- Add limitations and future research questions.
- Clean the repository and publish the code and report.

## 15. Repository structure

```text
agent-approval-policy-benchmark/
├── README.md
├── LICENSE
├── pyproject.toml
├── configs/
│   ├── llm_only.yaml
│   ├── baseline.yaml
│   ├── prompt_only.yaml
│   ├── llm_judge_gate.yaml
│   ├── policy_gate.yaml
│   ├── provenance_gate.yaml
│   └── hybrid_gate.yaml
├── policy/
│   ├── policy_v1.md
│   └── rules.py          # plain Python, no LangChain imports
├── scenarios/
│   ├── development.jsonl
│   ├── evaluation.jsonl
│   └── schema.json
├── agent/
│   ├── graph.py          # LangGraph: model → policy_gate → tools
│   ├── gate_node.py      # calls policy/rules.py, interrupt() on APPROVE
│   ├── llm_only.py       # condition 0, single model call
│   ├── tools.py          # langchain-core @tool definitions
│   └── state.py          # graph state with provenance tags
├── evaluation/
│   ├── run_experiment.py
│   ├── metrics.py
│   └── checks.py
├── results/
│   ├── raw/
│   ├── tables/
│   └── figures/
├── report/
│   ├── preregistration.md
│   └── technical_report.md
└── notebooks/
    └── analysis.ipynb
```

## 16. Acceptance criteria

The pilot is complete when:

- The same scenario suite runs under all seven conditions.
- The agent-versus-LLM comparison reports paired tests and confidence intervals for at least two models.
- Gate comparisons report clean accuracy, attacked accuracy, and consistency separately.
- Final evaluation labels were produced independently of the `rules.py` author, or the report states this as a limitation.
- Every tool call is logged with action, arguments, policy decision, and result.
- The policy engine can enforce `ALLOW`, `APPROVE`, and `DENY` independently of the model.
- The evaluation set contains documented gold labels and reasons.
- Results include security, utility, and operational metrics.
- At least one failure case is analysed for each configuration.
- The repository can be run by another person using the README.
- The report clearly states that this is a pilot study and identifies limitations.

## 17. Potential PhD extension

The pilot can develop into a larger research program on:

- Adaptive approval policies that learn from user feedback.
- Formal risk models for agent actions.
- Provenance-aware policies for RAG and repository content.
- Approval policies for multi-agent systems.
- Human workload and trust calibration.
- Policy guarantees for software-engineering agents.
- Cross-model and cross-framework evaluation.

## 18. Suggested title for the report

**Human Approval Policies for Tool-Using Software Agents: A Pilot Study of Security–Utility Trade-offs**

