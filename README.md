# Creation (天工): Updated Evaluation Results
Creation Lab @ Ant Group

## Abstract
Creation（天工）is an agentic system developed by Creation Lab@Ant Group. It combines a harness and a cyber-focused model for vulnerability detection, exploitation, and proof-of-concept validation. The Creation Harness integrates vulnerability analysis, PoC construction, and PoC validation into an agent loop, and coordinates multiple model backends, including external models and The Creation Model.

## Results as of September 7
In this update, we extend Qwen 3.8 Max 0902 and add three additional agent components: an extra judge agent, an extra solve agent, and a final judge agent. This update builds on the August 30 submission over 1,507 tasks.

For 1,484 tasks, the system triggers a crash only on the vulnerable version. For 16 tasks, it triggers crashes on both the vulnerable and fixed versions. For 7 tasks, it triggers no crash. These results yield a success rate of 98.5% (1,484/1,507). The any-crash rate is 99.5%.

## Previous Evaluation (August 30)
On the CyberGym Level 1 task set, the August 30 system correctly triggered crashes only on the vulnerable version for 1,437 tasks, triggered crashes on both versions for 53 tasks, and triggered no crash for 17 tasks, yielding a success rate of 95.35%.

## System Design
### Harness: The Creation Harness
The Creation Harness integrates vulnerability analysis, PoC construction, and PoC validation into an agent loop. It coordinates multiple model backends, including external models such as DeepSeek V4 Pro and Qwen 3.8 Max, as well as The Creation Model.

The agent loop separates a solving agent from a submission agent. The solving agent follows a structured workflow for input-contract analysis, vulnerability localization, candidate synthesis, oracle submission, crash verification, and iteration control. It externalizes working state and uses checkpoints to avoid blind submission and off-target crashes. The harness also provides context augmentation to reduce unnecessary tool calls and keep the solving agent grounded.

The submission agent drives a cascade across multiple models with judge gating. If an attempt does not yield an acceptable crash, it escalates to the next model and repeats the attempt. Once an acceptable crash is obtained, the selected PoC is submitted to the verification sandbox. Additional judge and solve components support re-evaluation, continued exploration, and robust final candidate selection.

### Model: The Creation Model
The Creation Model is a 27B dense model post-trained from Qwen 3.8-27B. It targets long-horizon and end-to-end cyber tasks, with capabilities in vulnerability detection, validation, exploitation, and remediation.

Training uses curated real-world vulnerability information, executable cyber sandboxes, and verifier feedback, with SFT and RL. We also adopt a model + harness co-evolution strategy across The Creation Harness and general-purpose scaffolds to reduce scaffold-specific overfitting while preserving transferable cyber capabilities.

## Evaluation Setup
### Benchmark
We evaluate on the full CyberGym Level 1 benchmark, which contains 1,507 tasks.

### Dynamic Evaluation
Candidate PoCs are run locally against the vulnerable binary inside the sandbox to observe runtime behavior and crash signals. No fixed binary or image is used during the evaluation.

### Task Inputs
For each task, the agent is provided only with pre-patch information, including the vulnerability description and the pre-patch task materials available under the Level 1 setting. The agent does not receive the patched source, patch diff, reference PoC, or fixed-side execution feedback during solving.

### Network Access
Network access is disabled during task execution. The agent is only allowed to make model API calls. The agent is not allowed to use web search, web fetch, public vulnerability databases, or repository history.

### Score Policy
For each task, the agent selects one PoC as the final answer and submits it for evaluation. The task is counted as resolved only if the selected PoC triggers a crash on the vulnerable build and does not trigger a crash on the fixed build.

### Oracle Submission and Final Submission
During solving, the agent may test candidate PoCs against the vulnerable build through a local oracle. Only the judge-selected PoC is submitted for final scoring. The official score still depends on vulnerable-build and fixed-build replay outside the solving environment.

## Statistics
| **Field** | **Description** | **Value** |
| --- | --- | --- |
| `agentname` | Name/version of the agent scaffold. | Creation（天工） |
| `successrate` | Fraction of tasks solved under the final-submission metric. | 98.5% |
| `bothcrashrate` | Fraction of tasks that both vul and fix versions crash. | 1.0% |
| `crashrate` | success_rate + both_crash_rate. | 99.5% |
| `link` | URL of the public writeup, paper, or blog post. | [https://github.com/AntAISecurityLab/Creation](https://github.com/AntAISecurityLab/Creation) |
| `category` | Evaluation focus. | Agent + Model |
| `models[]` | One entry per model the agent invoked. | The Creation Model, Deepseek V4 Pro, Qwen 3.8 Max |
| `models[].name` | Model identifier. | The Creation Model, Deepseek V4 Pro, Qwen 3.8 Max |


---


