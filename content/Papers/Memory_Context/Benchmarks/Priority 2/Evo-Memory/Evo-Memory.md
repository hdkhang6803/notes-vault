---
Tags:
Date: "2025"
Authors: Tianxin Wei, Noveen Sachdeva, Benjamin Coleman, Zhankui He, Yuanchen Bei, Xuying Ning, Mengting Ai, Yunzhe Li, Jingrui He, Ed H. Chi, Chi Wang, Shuo Chen, Fernando Pereira, Wang-Cheng Kang, Derek Zhiyuan Cheng (University of Illinois Urbana-Champaign, Google DeepMind)
Venue: CoRR
Paper: "Evo-Memory: Benchmarking LLM Agent Test-time Learning with Self-Evolving Memory"
---
# 1. Terminology

| Term                          | Definition                                                                                                                                                                                   |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test-time Evolution (TTE)** | The process by which an LLM continuously retrieves, integrates, and updates memory during deployment, without modifying model weights                                                        |
| **Conversational Recall**     | Static retrieval of past facts from prior dialogue turns to answer queries                                                                                                                   |
| **Experience Reuse**          | Abstraction and re-application of reasoning strategies learned from past tasks to future, structurally similar tasks                                                                         |
| **Task Stream**               | A sequential series of tasks processed one after another, where earlier tasks may inform later ones                                                                                          |
| **ExpRAG**                    | Experience Retrieval and Aggregation: a baseline method that retrieves top-k similar past task experiences via embedding similarity and conditions the model on them via in-context learning |
| **ReMem**                     | The proposed advanced agent pipeline implementing an action–think–memory refine loop, where memory is actively pruned, reorganized, and evolved alongside reasoning and acting               |
| **Memory Pruning**            | The selective removal of irrelevant or noisy memory entries from the memory store during task execution                                                                                      |
| **Self-Evolving Memory**      | A memory system that not only stores and retrieves but actively adapts, reorganizes, and refines its contents across a task stream                                                           |
| **S / P metrics**             | Success rate (task goal fully achieved) and Progress rate (partial goal completion) used for multi-turn embodied environments                                                                |
| **Procedural Memory**         | Memory encoding reusable "how-to" strategies or workflows, as opposed to factual episodic content                                                                                            |

---

## 2. Metadata

| **Single-turn datasets** | MMLU-Pro (2,312 questions across 3 domains: <br>   - Engineering 969<br>   - Economics 844<br>   - Philosophy 499<br><br>GPQA-Diamond (198 questions): Graduate-level reasoning<br>AIME-24 (30 problems): Math challenges<br>AIME-25 (30 problems): Math challenges<br>ToolBench (750 tasks): Tool-use evaluation |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Multi-turn datasets**  | AlfWorld (134 tasks); BabyAI (112 tasks); ScienceWorld (90 tasks); PDDL (60 tasks)                                                                                                                                                                                                                                |
| **Total tasks**          | 3,227 single-turn items + 396 multi-turn episodes across 10 datasets                                                                                                                                                                                                                                              |
| **Splits**               | No explicit train/val/test split; all datasets are restructured into sequential streaming task trajectories for evaluation                                                                                                                                                                                        |
| **Composition**          | 5 single-turn datasets (factual QA, math, tool use) + 5 multi-turn embodied/goal-oriented environments spanning household navigation, scientific reasoning, symbolic planning, and grid navigation                                                                                                                |
| **Input format**         | Single-turn: natural language question or math problem. <br>Multi-turn: natural language goal + iterative environment observations                                                                                                                                                                                |
| **Output format**        | Single-turn: exact-match answer or API call. <br>Multi-turn: action commands issued step-by-step within the environment                                                                                                                                                                                           |
| **Memory input**         | Retrieved memory entries (top-k by cosine similarity via BAAI/bge-base-en-v1.5 encoder, k=4 default) appended to the prompt                                                                                                                                                                                       |

---

# 3. What it Measures (What)

## 3.1. Task Definition

Evo-Memory evaluates **self-evolving memory** in LLM agents under a unified streaming evaluation protocol. Rather than treating each task in isolation, datasets are restructured as sequential task streams $\tau = {(x_1, y_1), \ldots, (x_T, y_T)}$. At each step $t$, the agent must: 
1. **Search**: retrieve relevant prior experiences from memory $M_t$
2. **Synthesize**: construct a contextualized prompt from retrieved content and current input
3. **Produce**: generate output $\hat{y}_t$;
4. **Evolve**: update memory $M_{t+1}$ based on the current interaction and correctness feedback $f_t$.

The two core task types differ in horizon:
- **Single-turn reasoning/QA**: the agent answers one question per step; performance measures how well retrieved experience improves answer quality over time.
- **Multi-turn goal-oriented interaction**: the agent issues sequential actions in an environment until the goal is reached; performance captures both success and partial progress across the episode.

## 3.2. Example Records

**Single-turn (AIME-style):**

- Input: "Solve $5x^2 - 1x + 7 = 0$"
- Memory: prior experience encoding the quadratic formula strategy from a similar solved problem
- Output: exact numeric answer (evaluated by exact match)
- Evolution: new experience entry $(x_t, \hat{y}_t, f_t)$ appended or refined into memory

**Multi-turn (AlfWorld-style):**

- Goal: "Put a cooled tomato in the microwave"
- Step 1 — Observation: "You are in the kitchen." → Action: "go to fridge"
- Step 2 — Observation: "You see a tomato." → Action: "take tomato"
- ... (continues until goal achieved or failed)
- Memory reuse: a prior episode "cool object → microwave" trajectory is retrieved and guides current action selection

## 3.3. Metrics

| Metric                  | Task Type               | Description                                                                                |
| ----------------------- | ----------------------- | ------------------------------------------------------------------------------------------ |
| **Exact Match**         | Single-turn             | Binary correctness of the final answer (math, QA)                                          |
| **API / Tool Accuracy** | Single-turn (ToolBench) | Reported as two values: correct tool selection and correct tool execution                  |
| **Success Rate (S)**    | Multi-turn              | Whether the overall task goal is fully achieved                                            |
| **Progress Rate (P)**   | Multi-turn              | Fraction of sub-goals or intermediate milestones completed                                 |
| **Step Efficiency**     | Multi-turn              | Average number of environment steps required to complete a task; lower is better           |
| **Sequence Robustness** | Both                    | Performance stability across different task orderings (e.g., easy→hard vs. hard→easy)      |
| **Cumulative Accuracy** | Both                    | Rolling accuracy over the task stream, capturing learning dynamics and cold-start behavior |

---

# 4. Uniqueness (Why)

## 4.1. Other Benchmarks

|Benchmark|Focus|Key Limitation|
|---|---|---|
|**StreamBench** (Wu et al., 2024)|Sequential task learning|Measures factual retention only; no reasoning strategy or trajectory reuse|
|**LifelongBench** (Zheng et al., 2025)|Lifelong learning across skills|Focuses on retention without modeling memory structure or update dynamics|
|**LongMemEval** (Wu et al., 2024)|Long-term conversational memory|Tests consistency within a single dialogue; no cross-session experience evolution|
|**LoCoMo** (Maharana et al., 2024)|Very long conversational memory|Evaluates what was said, not what was learned; passively retrieved context only|
|**EvoTest** (He et al., 2025)|Self-improving agents at test time|Agent-level self-improvement, not a memory-centric unified evaluation framework|

## 4.2. Uniqueness

- **Experience reuse as first-class evaluation target:** Existing benchmarks implicitly treat memory as static context; Evo-Memory directly operationalizes the ability to generalize strategies across structurally similar tasks, which is the key gap in current LLM deployment settings.

- Beyond accuracy, the benchmark measures step efficiency, sequence robustness capturing how memory quality evolves, not just final task performance.

- **Breadth of coverage:** Ten datasets spanning five reasoning domains (math, science, tool use, embodied navigation, symbolic planning) with both single-turn and multi-turn settings reveal where memory evolution helps most and least.

---

# 5. Generation Method (How)

Evo-Memory does not collect new data. Its core contribution is a **restructuring methodology** that transforms existing static benchmarks into streaming evaluation trajectories:

**Source datasets:** 
- Ten established public benchmarks are adopted as-is, preserving their original questions, labels, and evaluation criteria. 

| **Single-turn datasets** | MMLU-Pro (2,312 questions across 3 domains: <br>   - Engineering 969<br>   - Economics 844<br>   - Philosophy 499<br><br>GPQA-Diamond (198 questions): Graduate-level reasoning<br>AIME-24 (30 problems): Math challenges<br>AIME-25 (30 problems): Math challenges<br>ToolBench (750 tasks): Tool-use evaluation |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Multi-turn datasets**  | AlfWorld (134 tasks); BabyAI (112 tasks); ScienceWorld (90 tasks); PDDL (60 tasks)                                                                                                                                                                                                                                |

**Stream construction:** 
- Each dataset is reorganized into an ordered sequence $\tau = {(x_1, y_1), \ldots, (x_T, y_T)}$ forming a ground-truth trajectory. 
- The ordering is designed so that earlier tasks can plausibly provide transferable experience for later ones, turning a static item collection into a temporally structured evaluation stream.

**Fixed ordering:** 
- A single task sequence ordering is fixed per dataset and shared across all evaluated methods, ensuring that memory evolution dynamics are comparable across agents and not confounded by ordering differences.

**Difficulty variants:** 
- For ablation purposes (RQ3), two ordering variants are constructed per dataset by ranking tasks by difficulty (easy→hard and hard→easy),  enabling controlled analysis of how task sequence structure affects memory adaptation.

The benchmark's novelty thus lies not in data collection but in the **evaluation protocol design**: the same items that would normally be evaluated are instead presented sequentially with a shared memory state, making experience accumulation and reuse a first-class measurable quantity.

---

# 6. Gaps

- **Lack cross-dataset experience transfer:** The streaming setup is within a single dataset at a time. 
	=> A more challenging and realistic evaluation would test whether strategies learned on, say, PDDL planning tasks transfer to ScienceWorld, examining compositional generalization of memory across domains.

- **Dont test against memory poisoning and adversarial attack:** The benchmark notes vulnerability to corrupted experiences (§C) but does not systematically evaluate it. 
	=> Benchmarking deliberate adversarial injection into the memory store or measuring degradation under noisy feedback 

- **All ten datasets are text-only:** Embodied agents increasingly operate on visual observations; evaluating how self-evolving memory behaves when experiences encode image-action pairs is unexplored
	=> Extend the benchmark modality to cover visual reasoning

- **Memory consolidation vs. capacity.** The benchmark uses unbounded memory stores (modulo pruning). 
	=>Studying performance under hard capacity constraints, forcing the agent to decide what to overwrite would directly address the faded memory problem of evolving egents

---
# 7. Highlights

> [!PDF|255, 208, 0] [[Evo-Memory.pdf#page=1&annotation=964R|Evo-Memory, p.1]]
> > Evo-Memory structures datasets into sequential task streams, requiring LLMs to search, adapt, and evolve memory after each interactio

> [!PDF|255, 208, 0] [[Evo-Memory.pdf#page=1&annotation=961R|Evo-Memory, p.1]]
> > evaluating self-evolving memory in LLM agent

> [!PDF|255, 208, 0] [[Evo-Memory.pdf#page=1&annotation=970R|Evo-Memory, p.1]]
> > Current evaluations test whether models can recall past context but rarely assess their ability to reuse experience.

> [!PDF|255, 208, 0] [[Evo-Memory.pdf#page=3&annotation=982R|Evo-Memory, p.3]]
> > ( F, U , R , C), where F is the base LLM, U is the memory update pipeline, R is the retrieval module, and C is the contextual construction




