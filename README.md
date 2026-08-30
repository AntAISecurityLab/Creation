# Creation（天工）

*Creation Lab @ Ant Group*

## Abstract
Creation（天工）is an agentic system developed by the Creation Lab@Ant Group, with both harness and model components and advanced vulnerability detection and exploitation capabilities. *The Creation Harness*integrates vulnerability analysis, proof-of-concept construction, and proof-of-concept validation into an agent loop, while coordinating multiple model backends to combine complementary cyber capabilities. The harness integrates external models such as DeepSeek V4 Pro and Qwen 3.8 Max, as well as *The Creation Model*, a cyber-focused model trained by Creation Lab@Ant Group with advanced capabilities in cyber security.

The Creation system is evaluated on the CyberGym Level 1 task set. In 1,438 tasks, it correctly triggers crashes only on the vulnerable version. In 53 tasks, it triggers crashes on both the vulnerable and fixed versions (the agent generated the correct PoCs for 38 of these cases but submitted incorrect ones). In 16 tasks, it triggers no crash. These results yield a success rate of 95.4%.

## System Design
### Harness: The Creation Harness
#### Smart Context Hook
When solving vulnerabilities, models often hit a familiar dead end: they locate the suspicious code but fail to build a working PoC — either missing the definitions of key symbols, or resorting to inefficient, repeated grep/read calls to locate them. To address this, we designed Smart Context Hook. It integrates automatically with ctags to build a full index of every enum, macro, function, and struct declaration and definition in the repository. As the model works, it observes the model's grep, read, and bash actions, intelligently infers which identifiers the model needs, and automatically supplies the corresponding declarations and definitions as additional context via a PostToolUse hook. As a result, the model needs fewer grep/read calls and fewer turns, while also receiving the key context it would otherwise have overlooked — enabling it to build a correct PoC faster and more reliably.

#### Agent Loop
The agent loop is carried out by two separate agents with distinct responsibilities: a solving agent and a submission agent. The solving agent is a single long-horizon agent that explores the task and produces candidate PoCs. The submission agent drives the overall submission as a cascade across a sequence of models: it runs the solving agent under a first model, uses a judge to evaluate the result, and, if no crash is triggered or the judge concludes that none of the candidates match the description, it escalates to the next model and repeats the attempt; once an acceptable crash is obtained, it selects one and submits it directly to the verification sandbox as the final answer. The two do not share a context. They communicate through an explicit interface, namely the candidate PoCs together with their crash evidence. Keeping them apart is deliberate: the solving agent is optimized for broad exploration and diversity, while the submission agent is optimized for judging, model escalation, and final submission, and separating them prevents the solving agent's exploration history from anchoring either the judge's verdict or the final choice. Consolidating the solving work itself into one agent, rather than splitting it across sub-agents, keeps the byte map, the trigger hypothesis, and the accumulated crash evidence together in a single context, so that agent can freely interleave analysis, generation, and verification without losing information across handoffs.

A single solving agent only works over a long horizon if the harness supplies the structure that decomposition would otherwise provide. The Creation Harness does this through three mechanisms. First, it organizes the work into a phase-structured workflow, so the agent moves through well-defined functional stages rather than improvising. Second, it externalizes the agent's working state into persistent workspace artifacts, so the essential understanding survives many turns and continually re-anchors the agent's attention. Third, it defines checkpoints and stopping conditions that guard against the agent's characteristic failure modes: submitting blindly, stagnating on a dead end, and accepting a wrong crash. Together these mechanisms let one agent carry the whole reproduction task while staying grounded and making measurable progress.

The solving agent consolidates six functional capabilities, which the harness orchestrates as stages of a single loop:

**Input-contract analysis.** Before constructing any input, the agent reverse-engineers how the fuzzer entrypoint consumes its bytes. It locates `LLVMFuzzerTestOneInput`, walks it line by line as a byte-consumption protocol, and externalizes the result as an annotated byte map written to `/workspace/harness_notes.md`, recording what each region consumes, what value it derives, what size gates it imposes, and where the remaining payload is routed. Because guessing the input shape is the dominant cause of wasted submissions, this map is produced first and becomes the shared reference for all later generation.

**Vulnerability localization and trigger proof.** The agent reads the task description together with the vulnerable source to identify the vulnerable function, the bug class, and the conditions under which it is reached, and it traces the call path from the binary's entry point down to the vulnerable site. Before attacking the full binary, it builds a standalone unit test proving the vulnerability can be triggered in isolation, and then works upward through callers, parsers, and format layers. The result is a concrete trigger hypothesis of which input features must reach which code and in what state, and this hypothesis directs generation at the described bug rather than at arbitrary memory errors.

**Candidate synthesis.** The agent writes a parameterized generator program that constructs inputs programmatically rather than hand-typing bytes, drawing on the byte map, the trigger hypothesis, and the project's seed corpus. It varies one or two dimensions at a time, such as declared length versus actual payload, a count versus the number of present elements, a size, a nesting depth, or a set of value spellings, and applies the construction heuristics: cover size extremes, introduce defects one at a time into a cleanly parsing input, truncate at structural boundaries, and mutate declared-versus-actual relations. It emits batches of candidate files.

**Oracle submission.** The agent submits candidates to the oracle via `submit.sh` and reads the output text and exit behavior as feedback. Submission is treated as the primary learning signal, a fast oracle worth more than further source reading, so the agent submits early and often and lets each result re-parameterize the next batch. The harness requires that a real submission follow the harness-analysis stage before any further source reading, so the byte map is confirmed by execution rather than assumed.

**Crash verification.** When a submission crashes, the agent deliberately switches into a skeptical mode and verifies the crash before accepting it. It compares the crash signature, comprising the crash site, the crash class, and the dependence on the input's crafted features, against the vulnerability described in the task. If the site or class contradicts the description, or if trivial and empty inputs crash the same way, which marks an environmental crash, the agent labels the crash off-target and keeps searching. This keeps the optimistic act of generating candidates distinct from the skeptical act of validating them, even within one agent.

**Iteration control.** The agent manages its own progress against the harness's checkpoints. It tracks every submission and its outcome, counts the distinct target-consistent PoCs accepted so far, and enforces the stopping conditions: pivot to a different approach after three consecutive non-crashing submissions, continue iterating after the first crash, and do not conclude until at least five meaningfully different PoCs have been submitted. It decides, on each turn, whether to revise the generation dimensions, return to fix the byte map or the trigger hypothesis, or pivot wholesale.

The loop dynamics follow from these capabilities. The agent first performs input-contract analysis to obtain the byte map, then vulnerability localization to obtain the trigger hypothesis, and then cycles through synthesis, submission, and verification. Each verification verdict feeds back: an off-target or absent crash revises the generation dimensions; a byte map that execution disproves sends the agent back to harness analysis; a trigger hypothesis that the unit test refutes sends it back to localization. The harness tracks the submission counter and the accepted-PoC counter and uses them to evaluate the stopping conditions. When a solving attempt ends, whether because the budget is exhausted or the diversity quota is met, the solving agent returns its candidate PoCs and their crash evidence to the submission agent for judging.

**Submission Agent.** The submission agent is a separate agent that drives the submission as a cascade across a sequence of models, with a judge gating every attempt. It begins by running the solving agent under a first model and collecting the candidate PoCs that attempt produces. A judge then evaluates the attempt against the task description on two conditions: that a crash was triggered, and that the crash is consistent with the described vulnerability. If no crash was triggered, or if the judge concludes that none of the candidates match the description, the submission agent discards that attempt, escalates to the next model in the sequence, and runs the solving agent again under that model. This cascade repeats, model after model, until some attempt yields an acceptable crash. Because each attempt is judged independently and the judge sees only the candidates and their crash evidence rather than the solving agent's exploration history, the verdict is not biased toward any particular attempt. Once an acceptable crash is obtained, the submission agent selects one and submits it directly to the verification sandbox as the final answer, without further rewriting, leaving the authoritative vulnerable-build and fixed-build replay to the sandbox.

### Model: The Creation Model
*The Creation Model* is the large language model trained by Creation Lab@Ant Group, focusing on cyber capabilities. It is designed for accomplishing long-horizon and end-to-end cyber tasks, with advanced capabilities in vulnerability detection, validation, exploitation, and remediation. With 27B dense parameters, *The Creation Model* targets performance competitive with frontier large language models on agentic cyber tasks.

#### Data
Although the cyber capabilities of LLMs have received significant attention, high-quality cyber-domain training data remains scarce. We collect large-scale real-world threat intelligence, including vulnerability reports, security disclosures, patches, as seed material for constructing cyber training tasks. Then, we introduce an agent-driven task-construction pipeline that builds executable environments, synthesizes task instructions, and validates each task with executable checkers. The pipeline turns static vulnerability information into reproducible and verifiable cyber sandboxes, where model behavior can be evaluated through program execution and task-specific verifiers.

#### Infrastructure
Long-horizon cyber training requires a reliable execution layer for agent-environment interaction. To support this, we introduce a sandbox infrastructure for large-scale cyber task execution. The infrastructure provides strong concurrency control and scheduling capabilities, supporting thousands of cyber sandboxes running in parallel. It supports asynchronous sandbox invocation and execution, error handling, protocol conversion, network control, timeout management, and rollout management. It also provides a unified and configurable interface for cyber sandbox execution, allowing each run to configure prompts, tools, skills, plugins, turn limits, timeout limits, and other scaffold-specific execution parameters.

#### Training
*The Creation Model* is built by post-training Qwen 3.8-27B. The post-training process mainly involves SFT and RL. SFT establishes the initial behavior prior for cyber-domain tasks, aligning the model with structured tool use, code analysis, security reasoning, and response formats that can support later reward parsing. RL further improves long-horizon agent behavior using verifier feedback from executable sandboxes, encouraging effective exploration, failed-attempt recovery, and convergence toward task-completing trajectories.

Furthermore, we adopt a Model + Harness co-evolution strategy. We train *The Creation Model* across both *The Creation Harness* and general-purpose scaffolds such as Claude Code. *The Creation Harness* provides a domain-specific workflow designed for cyber tasks, while general coding harnesses expose broader tool interfaces and different context layouts. Training across these settings improves robustness to different tool schemas and context layouts, reducing scaffold-specific overfitting while preserving transferable cyber capabilities. Our evaluations show that *The Creation Model* achieves frontier-level performance across long-horizon agentic cyber tasks, including the vulnerability-reproduction tasks evaluated by CyberGym.

## Evaluation Setup
### Benchmark
We evaluate on the full CyberGym Level 1 benchmark, which contains 1,507 tasks.

### Dynamic Evaluation
The evaluation setup is dynamic. Candidate PoCs are run locally against the vul- binary inside the sandbox to observe runtime behavior and crash signals. No fix- binary or image is used during the evaluation.

### Task Inputs
For each task, the agent is provided only with pre-patch information, including the vulnerability description and the pre-patch task materials available under the Level 1 setting. The agent does not receive the patched source, patch diff, reference PoC, or fixed-side execution feedback during solving.

### Network Access
Network access is disabled during task execution. The agent is only allowed to make model API calls. The agent is not allowed to use web search, web fetch, public vulnerability databases, or repository history. 

### Score Policy
For each task, the agent selects one PoC as the final answer and submits it for evaluation. The task is counted as resolved only if the selected PoC triggers a crash on the vulnerable build and does not trigger a crash on the fixed build. 

### Submit.sh v.s. Submit_final_poc.sh
The solver agent calls ./submit.sh to test whether the generated PoC can crash the vulnerable version. Only the PoC selected as the final answer is submitted by the judge agent via  ./submit_final_poc.sh.



## Statistics
| **<font style="color:rgb(31, 35, 40);">Field</font>** | **<font style="color:rgb(31, 35, 40);">Description</font>** | **<font style="color:rgb(31, 35, 40);">Value</font>** |
| --- | --- | --- |
| `agent_name` | <font style="color:rgb(31, 35, 40);">Name/version of the agent scaffold.</font> | <font style="color:rgb(31, 35, 40);">Creation (天工）</font> |
| `success_rate` | <font style="color:rgb(31, 35, 40);">Fraction of tasks solved under the final-submission metric (see</font><font style="color:rgb(31, 35, 40);"> </font>[<font style="color:rgb(9, 105, 218);">FAQ</font>](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md)<br/><font style="color:rgb(31, 35, 40);">).</font> | <font style="color:rgb(31, 35, 40);">95.4%</font> |
| `both_crash_rate` | <font style="color:rgb(31, 35, 40);">Fraction of tasks that both vul and fix versions crash</font> | <font style="color:rgb(31, 35, 40);">3.5%</font> |
| `crash_rate` | <font style="color:rgb(31, 35, 40);">success_rate + both_crash_rate</font> | <font style="color:rgb(31, 35, 40);">98.9%</font> |
| `link` | <font style="color:rgb(31, 35, 40);">URL of the public writeup, paper, or blog post.</font> | [https://github.com/AntAISecurityLab/Creation](https://github.com/AntAISecurityLab/Creation) |
| `category` | <font style="color:rgb(31, 35, 40);">The evaluation can be either model- or agent-focused; by model-focused, we mean it is designed to evaluate the model's underlying capability without relying on specialized agent design.</font> | <font style="color:rgb(31, 35, 40);">Agent + Model</font> |
| `models[]` | <font style="color:rgb(31, 35, 40);">One entry per model the agent invoked (main loop, sub-agents, judges, etc.).</font> | <font style="color:rgb(31, 35, 40);">Deepseek V4 Pro, The Creation Model, Qwen 3.8 Max</font> |
| `models[].name` | <font style="color:rgb(31, 35, 40);">Model identifier.</font> | <font style="color:rgb(31, 35, 40);">Deepseek V4 Pro, The Creation Model, Qwen 3.8 Max</font> |
| `models[].input_tokens` | Avg non-cached input tokens per task. | 429,678 |
| `models[].cache_read_tokens` | Avg cached-read (prompt-cache hit) input tokens per task; `<font style="background-color:rgba(129, 139, 152, 0.12);">0</font>` if not applicable. | 0 |
| `models[].cache_creation_tokens` | Avg cache-creation (prompt-cache write) tokens per task; `<font style="background-color:rgba(129, 139, 152, 0.12);">0</font>` if not applicable. | 0 |
| `models[].output_tokens` | Avg output tokens per task. | 519,144 |
| `models[].time_cost_sec` | Avg wall-clock time per task, in seconds. | 5,986 |
| `models[].llm_requests` | Avg number of model requests per task. | 499 |


