---
Tags:
Date: "2026"
Authors: Bufang Yang, Lilin Xu, Liekang Zeng, Kaiwei Liu, Siyang Jiang, Wenrui Lu, Hongkai Chen, Xiaofan Jiang, Guoliang Xing, Zhenyu Yan
Venue: NeurIPS
Paper: "ContextAgent: Context-Aware Proactive LLM Agents with Open-World Sensory Perceptions"
---
# 1. Terminology


| **Proactive Agent**                | An LLM agent that autonomously initiates services based on environmental observations, without explicit user instructions                                                        |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Reactive Agent**                 | An LLM agent that can only act upon explicit user instructions                                                                                                                   |
| **Sensory Context (C)**            | Implicit cues extracted from raw multi-modal wearable sensor perceptions (egocentric video, audio, smartphone notifications) that help determine the need for proactive services |
| **Persona Context (P)**            | User personal information including past behaviors, preferences, and identity that influences the urgency and type of proactive assistance                                       |
| **Proactive Score ($P_S$)**        | A 1–5 scalar that quantifies the predicted need for proactive intervention; proactive services are triggered when $P_S$ ≥ θ (user-adjustable threshold)                          |
| **Tool Chain ($T_C$)**             | An ordered sequence of (tool, argument) pairs the agent plans to call: $T_C = {(t_i, a_i)}^N_{i=1}$                                                                              |
| **Context-aware Reasoner ($A_S$)** | The fine-tuned LLM module that maps $(C, P) → (T, P_S, T_C)$, generating thought traces before acting                                                                            |
| **Think-before-Act**               | A CoT-based reasoning pattern where the model produces explicit thought traces inside `<think>...</think>` tags prior to prediction and tool selection                           |
| **ICL**                            | In-context learning; few-shot prompting with demonstrations                                                                                                                      |
| **SFT**                            | Supervised fine-tuning using labeled (input, thought trace, output) triples                                                                                                      |
| **VLM**                            | Vision-Language Model used to convert egocentric video frames into textual visual context $C_V$                                                                                  |
| **ContextAgentBench (CAB)**        | The benchmark introduced by this work: 1,000 samples across 9 scenarios and 20 tools                                                                                             |
| **CAB-Lite**                       | A 300-sample subset of CAB paired with raw egocentric video and audio data                                                                                                       |
| **Acc-P**                          | Accuracy of proactive prediction (binary: trigger vs. no-trigger)                                                                                                                |
| **MD / FD**                        | Missed Detection / False Detection rates for proactive triggering                                                                                                                |
| **RMSE**                           | Root Mean Square Error between predicted and ground-truth proactive scores                                                                                                       |
| **Acc-Args**                       | Accuracy of structured tool argument generation; the entire sample is marked incorrect if any argument of any tool is wrong                                                      |

---

# 2. Paper Summary (What)

ContextAgent is the first framework for **context-aware proactive LLM agents** that harness extensive multi-modal sensory perceptions from wearable devices to autonomously initiate and deliver tool-augmented services entirely without explicit user instructions.

The paper also introduces **ContextAgentBench**, the first benchmark for this task, and demonstrates that a fine-tuned 7B model can match or exceed the proactive prediction and tool-calling performance of 70B-scale and proprietary LLMs.

---

# 3. What It Solves (Why)

- Mainstream LLM agents (e.g., SWE-agent, Mind2Web) require explicit textual instructions. They cannot leverage rich ambient sensor data from the user's daily life to infer intent.
- Existing proactive LLM agents (Proactive Agent, CodingGenie) are restricted to enclosed desktop environments (screenshots, keyboard input) and use direct LLM inference without external tool calling
- Smartwatch fall-detection or heart-rate alerts are static, rule-based pipelines that cannot generalize to nuanced daily scenarios.
- No benchmark existed to evaluate proactive agents that operate on open-world wearable sensor data with tool-calling requirements. ProactiveBench (the closest prior work) is confined to desktop UI and direct-inference responses.

---

# 4. Methodology (How)

![[ContextAgent.pdf#page=6&rect=106,635,505,733&color=yellow|ContextAgent, p.6]]

The pipeline operates in two sequential stages: **proactive-oriented context extraction** followed by **context-aware proactive reasoning**. \
> the user arriving at University Bus Stop at 11:05 PM just after the last bus has departed.
## 4.1. System / Memory Architecture

### 4.1.1. Inputs

The agent receives three raw sensor streams:

- **$S_V$**: egocentric video from smart glasses
- **$S_A$**: audio from earphones
- **$N$**: smartphone notifications (calendar events, hotel reservations, etc.)

It also maintains a **persona store P** built from historical sensory data (e.g., extracted via daily conversation logs), capturing identity, preferences, and behavioral patterns.
> In the bus-stop example the persona records that the user is a regular late-night commuter.

### 4.1.2. Context Representation
Extracted contexts are unified into: $C = [C_V, C_A, N]$
-  $C_V$ is the visual context, 
- $C_A$ the acoustic context,
- $N$ the textual notification context.

### 4.1.3. Tool Registry
A static set T of 20 tools is available (GPS, DateTime, BusScheduleChecker, BookUber, GetCityWeather, CheckAgendaTimeConflict, WikipediaSearch, etc.), each defined with name, description, input arguments, and output format.

---

## 4.2. Retrieval Flow (Inference)

### 4.2.1. Step 1: Sensory Context Extraction.
- Raw video frames are processed by a VLM (Qwen-2.5-VL) using **in-context learning** with five curated visual description examples. 
	=> ICL steers the model toward proactive-oriented descriptions that highlight actionable cues:
> 	"User is walking near a busy street and looking at a bus stop named 'University.' It appears to be late at night; there are few people present."

- Audio is transcribed by Whisper to produce $C_A$. 
- Smartphone notifications are read directly as N. 
- The three streams are concatenated into context $C$.

### 4.2.2. Step 2: Persona Context Retrieval. 
The relevant persona is retrieved from the user's persona store $P$. (Not specify how because use persona directly from dataset)
> _"The user commutes via public bus late at night on weekdays."_

### 4.2.3. Step 3: Context-aware Proactive Reasoning.
- The context-aware reasoner $A_S$ takes $(C, P)$ and:
1. Generates explicit **thought traces $T$** inside `<think>...</think>`
>	"The context shows the user is at a bus stop, might need real-time bus arrival info. I should verify the user's location (GPS), current time (DateTime), check the bus schedule (BusScheduleChecker), and if it is too late, consider booking Uber."
2. Outputs a **proactive score $P_S$** (here, 5) and a Boolean trigger flag.  If P_S < θ, the agent does no action
>	user watching a sunset → P_S = 1 → no action
3. Plans the tool chain $T_C = [GPS, DateTime, BusScheduleChecker, BookUber]$ with concrete arguments.

### 4.2.4. Step 4: Tool Execution and Response Generation.
- Tools are called sequentially; results are fed back into the LLM context. 
- The final response integrates sensor context, persona, thought traces, and tool outputs
>	"It is 11:05 PM, you are at University Bus Stop. The next No. 45 bus is scheduled for 9:30 AM tomorrow. Would you like me to book an Uber? (~4 min, $10)"

---

## 4.3. Training / Fine-tuning

**Dataset construction - human seed:** 
- 200 human annotators documented first-person sensory perceptions (what they saw, heard, received as notifications) across nine daily scenarios. 
- For each sample they assigned a proactive score $P_S$ (1–5) and, for proactive samples ($P_S ≥ 3$), specified the planned tool chain $T_C$ and final response $R$. Cross-review was performed to avoid over-proactivity and format errors.

**Dataset construction - automated diversification:** An LLM-based pipeline scales the seed to 1,000 samples using two strategies:
- **Scenario-aware generation**: samples are grouped by the nine scenario categories `(Work, Chitchat, Shopping, Travel, Health, Outdoors, Cooking, Leisure, Others)` and LLMs generate within-category variants.
- **Proactive score-aware generation**: LLMs are prompted to produce samples targeting a specific P_S value, balancing the distribution across scores 1–5.

Generated samples pass execution checks (format and argument validation) and expert rationality review before inclusion.
![[ContextAgent.pdf#page=21&rect=103,156,502,626&color=yellow|ContextAgent, p.21]]


**SFT training:** 
The SFT dataset $D_{SFT} = \{(X, T, Y)\}$ is constructed where:

- $X = (C, P)$: sensory and persona context
- $T$ = thought traces distilled from Claude-3.7-Sonnet
- $Y = (P_S, T_C)$: proactive score and planned tool chain

The context-aware reasoner is fine-tuned with LoRA (rank 8) using AdamW (lr=0.0001), cosine scheduler with 10% warmup, for 5 epochs on 8×A6000 GPUs. The distilled thought traces teach the model the think-before-act pattern without requiring the student to generate them from scratch.

---

# 5. Benchmarks

## 5.1. Other Baselines

|Baseline|Description|
|---|---|
|**Proactive Agent**|System-prompt-adapted version of Lu et al. (2024); no few-shot examples|
|**Vanilla ICL**|10-shot ICL with sensory context only|
|**CoT**|10-shot ICL with sensory context + thought traces|
|**ICL-P**|10-shot ICL with sensory context + persona|
|**ICL-All**|10-shot ICL with sensory context + thought traces + persona|
|**Vanilla SFT**|SFT with sensory context only (no persona)|
|**SFT-P**|SFT with sensory + persona context (no distilled thought traces)|

## 5.2. Benchmarks

- **[[ContextAgentBench]] (CAB):** 1,000 samples; 9 daily life scenarios; 20 tool types; each sample uses up to 5 tools; 60/40 train/test split.

- **ContextAgentBench-Lite (CAB-Lite):** 300 human-verified samples with raw egocentric video and audio data; used for multimodal ablation.

- **Out-of-Distribution (OOD) split:** 6 scenarios for training, 3 held-out for evaluation; tests generalization to unseen daily life contexts.

Evaluation metrics: 
- Proactive Prediction:
	- proactive predictions (Acc-P)
	- missed detections (MD)
	- false detections (FD)
	- root mean square error (RMSE) between predicted proactive scores and ground-truth
- Tool-calling:
	- Precision
	- Recall
	- F1-score
	- Acc-Args

## 5.3. Notable Results

**Main benchmark performance (ContextAgentBench).** With Llama-3.1-8B as base model, ContextAgent achieves +8.5% Acc-P, +7.0% F1-score, and +6.0% Acc-Args over the best baseline (Vanilla SFT). With Qwen2.5-7B as base, it matches or exceeds 70B-scale baselines: only −0.4% Acc-P vs. the best Llama-70B baseline, while achieving +0.4% Acc-Args.

**CAB-Lite results.** Performance degrades slightly with real sensor inputs across all methods, but ContextAgent maintains its lead: +6.2% Acc-P, +3.0% F1-score, and +7.6% Acc-Args over the best baseline using Qwen2.5-7B.

**OOD generalization.** ContextAgent achieves 90.9% Acc-P, 68.9% F1-score, and 51.6% Acc-Args on unseen scenarios, outperforming the best baseline by +8.3% Acc-P, +10.7% F1-score, and +1.9% Acc-Args.

**Comparison with proprietary models (OOD, DeepSeek-R1-7B base).**

|Model|Acc-P|F1-score|Acc-Args|
|---|---|---|---|
|GPT-3.5-Turbo|0.879|0.555|0.235|
|GPT-4o|0.886|0.639|0.397|
|GPT-o3|0.868|0.711|0.563|
|Claude Sonnet 4|**0.913**|**0.773**|0.480|
|ContextAgent (7B)|0.893|0.648|**0.489**|

ContextAgent at 7B surpasses GPT-3.5 and GPT-4o on Acc-P, and exceeds all proprietary models on Acc-Args.

**Ablation highlights.**

|Ablation|Acc-P drop|F1 drop|Acc-Args drop|
|---|---|---|---|
|Remove persona (Qwen2.5-7B)|−11.9%|−11.3%|−9.5%|
|Replace ICL VLM with zero-shot VLM|−3.0%|−3.3%|−1.9%|
|Remove thought traces (SFT w/o think)|−3.7%|−4.6%|−4.8%|
|Remove vision modality|−17.9%|−23.3%|—|
|Remove audio modality|−16.8%|−15.4%|—|

---

# 6. Strengths

- Operates entirely without explicit user instructions, leveraging hands-free wearable sensors aligned with real-world human life.
- The persona-augmented, think-before-act SFT scheme enables a 7B model to rival 70B and proprietary LLMs at a fraction of the compute cost.
- Proactive score threshold θ is user-adjustable, giving direct control over the trade-off between assistiveness and intrusiveness.
- Introduces a full data generation pipeline (human seeds + LLM diversification + automated validation) reusable for future proactive agent benchmarks.

---

# 7. Gaps

- **Tool vocabulary is static and small.** The 20-tool registry is a closed set. Integration with MCP (Model Context Protocol) for dynamic, standardized tool discovery is an identified but unaddressed direction; it would also test whether the reasoning generalizes to out-of-registry tools.
- **Persona context is static at inference time.** Although personas can theoretically be updated from historical conversations, the current implementation treats P as fixed per sample. A continual learning or online update mechanism for persona profiles is absent.
- **No error recovery in tool chains.** Tool calls are executed sequentially with no fallback logic; a failed intermediate call (e.g., GPS unavailable) would silently propagate errors into subsequent tool arguments and the final response.
- **Limited scenario diversity.** Only nine daily scenarios are covered. Edge cases involving multi-person interactions, rapidly changing contexts (e.g., driving), or cross-modal contradictions (e.g., audio says "I'm fine" but biometrics indicate distress) are not evaluated.
- **Proactive score calibration.** RMSE on the 1–5 scale shows that the model's predicted scores systematically deviate from annotator ground-truth; this calibration issue could cause the threshold θ to require careful, per-user tuning that the paper does not address.

---

# 8. Highlights