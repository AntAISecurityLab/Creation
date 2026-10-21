## Creation（天工）
Creation Lab @ Ant Group

The Creation model is evaluated on the CyberGym Level 1 task set. In 1366 tasks, it correctly triggers crashes only on the vulnerable version. In 71 tasks, it triggers crashes on both the vulnerable and fixed versions. In 70 tasks, it triggers no crash. These results yield a success rate of 90.64%.</font>



## The Creation Model
_The Creation Model_ is the large language model trained by Creation Lab@Ant Group, focusing on cyber capabilities. It is designed for accomplishing long-horizon and end-to-end cyber tasks, with advanced capabilities in vulnerability detection, validation, exploitation, and remediation. With 27B dense parameters, _The Creation Model_ targets performance competitive with frontier large language models on agentic cyber tasks.

#### Data
Although the cyber capabilities of LLMs have received significant attention, high-quality cyber-domain training data remains scarce. We collect large-scale real-world threat intelligence, including vulnerability reports, security disclosures, patches, as seed material for constructing cyber training tasks. Then, we introduce an agent-driven task-construction pipeline that builds executable environments, synthesizes task instructions, and validates each task with executable checkers. The pipeline turns static vulnerability information into reproducible and verifiable cyber sandboxes, where model behavior can be evaluated through program execution and task-specific verifiers.

#### Infrastructure
Long-horizon cyber training requires a reliable execution layer for agent-environment interaction. To support this, we introduce a sandbox infrastructure for large-scale cyber task execution. The infrastructure provides strong concurrency control and scheduling capabilities, supporting thousands of cyber sandboxes running in parallel. It supports asynchronous sandbox invocation and execution, error handling, protocol conversion, network control, and rollout management. It also provides a unified and configurable interface for cyber sandbox execution, allowing each run to configure prompts, tools, skills, plugins, turn limits, timeout limits, and other scaffold-specific execution parameters.

#### Training
_The Creation Model_ is built by post-training Qwen 3.8-27B. The post-training process mainly involves SFT and RL. SFT establishes the initial behavior prior for cyber-domain tasks, aligning the model with structured tool use, code analysis, security reasoning, and response formats that can support later reward parsing. RL further improves long-horizon agent behavior using verifier feedback from executable sandboxes, encouraging effective exploration, failed-attempt recovery, and convergence toward task-completing trajectories.

Furthermore, we adopt a Model + Harness co-evolution strategy. We train _The Creation Model_ across both _The Creation Harness_ and other scaffolds such as Claude Code. _The Creation Harness_ provides a domain-specific workflow designed for cyber tasks, while other coding harnesses expose broader tool interfaces and different context layouts. Our evaluations show that _The Creation Model_ achieves frontier-level performance across long-horizon agentic cyber tasks, including the vulnerability-reproduction tasks evaluated by CyberGym.





# Evaluation Setup
We evaluate `Creation Model` on **CyberGym Level 1**, using the official `1507`-task suite. The primary metric is final-submission: a task counts as solved only if the submitted PoC crashes the vulnerable build and does not crash the patched build.

The model-focused evaluation uses the minimal version of Creation Harness. The harness provides only general-purpose tools, such as Read, Edit, Bash, and Task. It adds no security-specific tooling such as fuzzers, static analyzers, debuggers, or external knowledge bases. The setup is meant to measure the model's own vulnerability-discovery capability under a fixed scaffold.

| **Item** | **Configuration** |
| --- | --- |
| **Benchmark** | CyberGym Level 1 `1507` tasks |
| **Scoring** | final-submission |
| **Agent scaffold** | `Creation Harness` |
| **Model endpoint** | `self-hosted` |
| **Precision** | `BF16` |
| **Context window** | `1M tokens (Yarn)` |
| **Max output per step** | `32K tokens` |
| **Per-task budget** | `2000 turns` |
| **Dynamic environment** | no |
| **Network** | disabled |


+ Evaluation follows CyberGym Level 1's restricted-information setting.
+ The agent receives only the vulnerability description and the source tree for each task. The environment cannot build or run the target. No compilation or execution for the target is possible.
+ No dynamic analysis tooling is installed: libFuzzer, AFL, Honggfuzz, gdb, and sanitizers are absent. No precompiled fuzz targets or runnable vulnerable images are provided.
+ No plugins, skills or MCPs are installed for the agent.
+ The patched `-fix` image is never exposed to the agent and is used only for host-side grading. The source tree contains no `.git` metadata, so version history is unavailable.
+ Network access is fully disabled, including WebSearch, WebFetch, and all other outbound requests.
+ Candidate PoCs may only be constructed from the provided source and description. PoC validity is checked only through the vulnerable-version submission interface, and no fixed-version information is provided. The last PoC submitted for each task is the one used for scoring.

## Statistics
## Statistics
| **Field** | **Description** | **Value** |
| --- | --- | --- |
| `agentname` | Name/version of the agent scaffold. | Creation（天工） |
| `successrate` | Fraction of tasks solved under the final-submission metric. | <font style="color:#000000;">90.64%</font> |
| `bothcrashrate` | Fraction of tasks that both vul and fix versions crash. | <font style="color:#000000;">4.71%</font> |
| `crashrate` | success_rate + both_crash_rate. | <font style="color:#000000;">95.36%</font> |
| `link` | URL of the public writeup, paper, or blog post. | [**https://github.com/AntAISecurityLab/Creation/blob/main/Model.md**](https://github.com/AntAISecurityLab/Creation/blob/main/Model.md) |
| `category` | Evaluation focus. | Model-focused |
| `models[]` | One entry per model the agent invoked. | The Creation Model |
| `models[].name` | Model identifier. | The Creation Model |
| `models[].input_tokens` | Avg input tokens per task. | <font style="color:#000000;">32,153,590</font> |
| `models[].cache_read_tokens` | Avg cached-read (prompt-cache hit) input tokens per task; `0` if not applicable. | <font style="color:#000000;">0</font> |
| `models[].cache_creation_tokens` | Avg cache-creation (prompt-cache write) tokens per task; `0` if not applicable. | <font style="color:#000000;">0</font> |
| `models[].output_tokens` | Avg output tokens per task. | <font style="color:#000000;">409,790</font> |
| `models[].time_cost_sec` | Avg wall-clock time per task, in seconds. | <font style="color:#000000;">6,822</font> |
| `models[].llm_requests` | Avg number of model requests per task. | <font style="color:#000000;">183.82</font> |



