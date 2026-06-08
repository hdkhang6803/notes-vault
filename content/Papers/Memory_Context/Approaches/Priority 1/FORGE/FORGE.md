---
Tags:
Date: "2026"
Authors: Igor Bogdanov, Chung-Horng Lung, Thomas Kunz, Jie Gao, Adrian Taylor, Marzia Zaman
Venue: CAIS
Paper: "FORGE: Self-Evolving Agent Memory With No Weight Updates via Population Broadcast"
Memory type:
  - Token-level
Agent env:
  - Population
  - Multi-agent
Record format: Text
Memory architecture:
  - 2-tier
Tackle Module: Memory management
Need offline initialization: true
Fine-tuning?: false
Other tags:
---
# 1. Terminology

| Term                     | Definition                                                                                                                                                                                                                                           |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **FORGE**                | Failure-Optimized Reflective Graduation and Evolution: the proposed staged, population-based protocol for evolving prompt-injected memory                                                                                                            |
| **ReAct**                | Reasoning + Acting: a prompting framework where agents interleave thought, tool use, and observation steps                                                                                                                                           |
| **Reflexion**            | A prompt-only self-improvement baseline where agents convert failed trajectories into verbal critiques stored in memory                                                                                                                              |
| **POMDP**                | Partially Observable Markov Decision Process: a sequential decision-making framework where the agent cannot fully observe the environment state                                                                                                      |
| **CybORG CAGE-2**        | A stochastic cyber-defense simulation environment used as the evaluation benchmark; a blue defender protects a 13-host network against an automated red attacker over 30 steps                                                                       |
| **B_line attacker**      | The scripted red agent in CAGE-2 following a kill chain: Discovery → Access → Lateral Movement (Move deeper into the system) → Privilege Escalation (Gain more access based on original foothold privilege)                                          |
| **Instance**             | One copy of the hierarchical ReAct agent hierarchy running in parallel within FORGE's population                                                                                                                                                     |
| **Planner**              | Top-level ReAct agent that selects the final defense action each step                                                                                                                                                                                |
| **Analyst**              | On-demand sub-agent that interprets host-level observations                                                                                                                                                                                          |
| **ActionChooser**        | On-demand sub-agent that ranks valid actions with justification                                                                                                                                                                                      |
| **Reflector**            | Learning agent that synthesizes failed trajectories into conditional heuristic rules (Rules representation)                                                                                                                                          |
| **Exemplifier**          | Learning agent that synthesizes failed trajectories into structured few-shot demonstrations (Examples representation)                                                                                                                                |
| **Rules**                | Memory artifact type: ordered lists of conditional heuristics (if-then form) injected into agent prompts                                                                                                                                             |
| **Examples**             | Memory artifact type: structured ReAct-style demonstrations (Thought–Tool–Observation–Answer) injected as few-shot context                                                                                                                           |
| **Mixed**                | Memory artifact type: both Rules and Examples generated over the same failure context                                                                                                                                                                |
| **Champion Broadcast**   | The outer-loop mechanism that copies the best-performing instance's memory to all other active instances after each stage                                                                                                                            |
| **Graduation**           | Early stopping: instances whose checkpoint return exceeds threshold θ are frozen and excluded from further training                                                                                                                                  |
| **Checkpoint**           | A frozen single-episode probe used during training for champion selection and graduation decisions                                                                                                                                                   |
| **PBT**                  | Population-Based Training: the RL framework FORGE adapts to the prompt/text space                                                                                                                                                                    |
| **Dynamic Memory**       | Initially empty memory slots per agent that accumulate learned artifacts during training                                                                                                                                                             |
| **Persistent Memory**    | Static instructions and domain knowledge set by the user, not modified during training                                                                                                                                                               |
| **Failure Trigger (τ)**  | Per-step reward threshold below which an episode is aborted and reflection is invoked                                                                                                                                                                |
| **Step**                 | One action taken by the agent in the environment (e.g., Analyse host_3). CAGE-2 episodes run for up to 30 steps.                                                                                                                                     |
| **Attempt**/ **Episode** | One full episode run (up to 30 steps) under the current memory state. If a step reward falls below τ, the attempt is aborted early. Either way, one attempt = one episode run.                                                                       |
| **Stage**                | A block of up to k_A = 3 attempts per instance. All instances run their attempts independently and in parallel within a stage. The stage ends when all instances finish their attempts, triggering checkpoint evaluation, graduation, and broadcast. |
| **Session**              | One complete run of the full FORGE protocol: S = 6 stages × k_A = 3 attempts × N = 10 instances. This is the unit reported in Tables 13–17 (e.g., "7 independent sessions for Gemini").                                                              |



---

# 2. Paper Summary (What)

FORGE proposes a gradient-free protocol for improving LLM-based sequential decision-making agents purely through prompt-injected memory, with no weight updates and no distillation from a stronger teacher model. 
The core idea is to run a population of $N$ hierarchical ReAct agents in parallel over $S$ stages, where each agent accumulates natural-language knowledge artifacts (Rules, Examples, or both) from its own failed trajectories via a Reflexion-style inner loop. After each stage, the best-performing instance's memory is broadcast to all others (champion broadcast), and instances that exceed a performance threshold are frozen and removed from further training (graduation). 

---

# 3. What it Solves (Why)

Three open problems motivate FORGE in the context of prompt-only adaptation for stochastic, long-horizon decision-making:
- Prior self-improvement systems commit to a single artifact type (heuristics or examples) without controlled comparison.
- Isolated Reflexion (single-stream reflection) lacks selection pressure: individual instances can accumulate counterproductive artifacts that degrade performance below zero-shot, and even good instances produce high-variance policies.

---

# 4. Methodology (How)

![[FORGE.pdf#page=3&rect=52,507,561,709&color=yellow|FORGE, p.3]]
>**Toy Task: Navigating a 5-room maze** The agent must reach Room 5 from Room 1 in 10 steps. Each step it chooses a door (Left, Right, Stay). It gets −1 for hitting a wall, −0.5 for backtracking, and 0 for each step closer to Room 5. Episode return = sum over 10 steps (best possible = 0).

FORGE has three nested components: 
- a hierarchical agent architecture
- an inner Reflexion loop
- an outer population protocol.

## 4.1. System architecture
- Each ReAct agent instance is a hierarchy of three role-specific ReAct agents sharing the same LLM: 
	- **Planner**: selects final defensive action <span style="color:rgb(0, 112, 192)">("go right towards Room 5")</span>
	- **Analyst**: interprets host-level observations<span style="color:rgb(0, 112, 192)"> ("door on left is locked")</span>
	- **ActionChooser**: ranks valid actions <span style="color:rgb(0, 112, 192)">("Choose: Right door")</span>

| Environment input                                                                                                                           | Analyst output                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| { <br>    "room_id": 3, <br>	"exits": {"L": 0, "R": 1}, <br>	"visited_rooms": [1, 2, 3], <br>	"recent_rewards": [-0.5, 0, -1.0, -0.5] <br>} | { <br>     "left_exit": "locked", <br>	 "right_exit": "open", <br>	 "loop_detected": false, <br>	 "wall_hit_recently": true, <br>	 "recommendation": "go Right" <br>} |
- Each agent in the hierarchy maintains two memory slots:
    - **Persistent memory**: static user-supplied knowledge <span style="color:rgb(0, 112, 192)">("Room 5 is always at the far right; walls give −1 penalty")</span>; never modified during training
    - **Dynamic memory**: initially empty; accumulates learned artifacts across attempts and stages
- Artifacts in dynamic memory take one of three forms: 
	- **Rules**: conditional heuristics
		> Planner rules: If Analyst reports current room has only one visible exit → plan to take that exit immediately, do not explore

	- **Examples**: structured ReAct demonstrations: Thought → Tool → Observation → Answer
		>- Thought: I am in Room 2. My goal is Room 5 which is to the right. Analyst should check the current exits before I commit to a direction. 
		>- Tool: get_analyst_report: {"current_room": "Room2", "step": 3, "recent_rooms": \["Room1", "Room2"\]} 
		>- PAUSE 
		>- Observation: \[ {"exit": "Left", "status": "locked", "confidence": 0.99}, {"exit": "Right", "status": "open", "confidence": 0.95}, {"exit": "Stay", "status": "always_available"} \] 
		>- Thought: Analyst confirms left is locked. Choosing Left would give −1 penalty. Right is open and moves toward Room 5. Choose Right. 
		>- Answer: Right 

	- **Mixed** (both)
- Dynamic<span style="color:rgb(0, 0, 0)"> memory has a fix</span>ed capacity cap to prevent unbounded context growth; artifacts are stored on disk between attempts
- N = 10 identical copies of this hierarchy run in parallel, each called an **instance**, each with its own independent memory state

## 4.2. Retrieval Flow

- At the start of every attempt, each instance's current dynamic memory is loaded from disk and **injected into the system prompt** of each agent in the hierarchy
- Sub-agents (Analyst, ActionChooser) are instantiated on-demand within the Planner's reasoning loop, each receiving their own role-specific prompt plus their injected memory slice
- During the attempt, the agent acts purely from this injected context; the context is static within an attempt
>At the start of Attempt 2 in Stage 1
>- instance_7 has one rule already in dynamic memory from its previous failed attempt: "If Analyst reports left door locked → ActionChooser should pick Right immediately." 
>- This rule is injected into all three agents' prompts before step 1, and the agents act on it from the very first step
## 4.3. Updating flow: within a stage
- Agent executes actions step-by-step; if per-step reward drops below τ = −1.1, the attempt is **aborted immediately**
- Full trajectory up to the failure point is captured and passed to a dedicated learning agent
- Reflector and Examplifier generate a knowledge artifact based on the chosen **representation**:
    - _Rules_: conditional heuristics synthesized by the **Reflector**
	- _Examples_: structured ReAct demonstrations synthesized by the **Exemplifier** (Thought → Tool → Observation → Answer)
    - _Mixed_: both agents run over the same failure context
- Artifact is appended to the instance's dynamic memory
- Attempt restarts from step 0
- Cycle repeats for up to $k_A$ = 3 attempts per stage

>- instance_7 hits the left wall at step 3 (reward −1.0 < τ). 
>- Trajectory captured: "Planner said go left, Analyst saw no lock warning, ActionChooser chose Left → wall." 
>- Reflector appends a new rule: "When room observation shows no visible exit on left → default to Right." 
>- Attempt restarts from step 1, now with two rules injected
>- This repeats up to k_A = 3 attempts per stage
## 4.4. Updating flow: between stages
![[FORGE.pdf#page=4&rect=55,537,565,707&color=yellow|FORGE, p.4]]
- N = 10 instances run their inner loops in parallel for S = 6 stages
- After each stage (all instances complete their inner loops), a **frozen checkpoint evaluation** produces a return $R_i$ for each instance
- Two mechanisms then fire in order:
    - **Graduation**: instances with $R_i$ > θ = −15 are frozen and excluded from all subsequent stages (saves compute, prevents regression)
	>	- instance_3 reaches Room 5 in 6 steps with return -1.0 (only two slight backtrack penalties) → crosses threshold → frozen

    - **Champion Broadcast**: the non-graduated instance with the highest $R_i$ has its **complete memory state copied** to all other active instances (full replacement, not interpolation)
>		- instance_5 has the best non-graduated checkpoint (-3.5). 
>		- Its dynamic memory (containing "always check Analyst for lock status before ActionChooser acts" and "if backtracking detected → Planner should skip Stay action") overwrites whatever instances 1, 2, 4, 6, 7, 8, 9, 10 had. 
>		- Stage 2 begins with all 9 active instances starting from instance_5's two rules, then diverging through their own failures in freshly seeded mazes

- Next stage begins with all active instances starting from the champion's memory, then independently diverging via their own failure trajectories
- After all stages complete, all instances (graduated or active) undergo a final frozen **post-session evaluation** (separate from checkpoints, used for all reported metrics)
- Setting `condition = Reflexion` disables broadcast, reducing the protocol to parallel independent Reflexion
---
# 5. Benchmarks

## 5.1. Other Baselines

N/A. The paper justifies this by arguing other methods would require non-trivial adaptation to work in CAGE-2
## 5.2. Benchmarks

| Benchmark                            | Description                                                                                                      |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| **CybORG CAGE-2 (B_line, 30 steps)** | Primary evaluation environment; stochastic POMDP, 13-host network, scalar per-step reward, partial observability |
**The network** has 13 hosts across 3 subnets:

```
Subnet 1 (User)          Subnet 2 (Enterprise)       Subnet 3 (Operational)
─────────────────        ─────────────────────        ──────────────────────
User Hosts ×5            Enterprise Hosts ×4          Op Hosts ×3
	                                                  Op Server ×1
```

**The red agent (B_line)** already knows the network topology and follows a fixed kill chain — it does not explore randomly, it beelines straight for the Operational Server:

```
Start → Discover Subnet 1 → Exploit User Host → Lateral Move to Enterprise 
      → Exploit Enterprise Host → Reach Op Server → Impact (deal damage)
```

**The blue agent** observes only partial signals (host-level alerts, process anomalies) and must choose one action per step from: Monitor, Analyse, Remove, Restore, or Decoy. It cannot directly see the attacker's position — it has to infer it.

An **episode** is 30 steps. The blue agent wins by keeping the attacker away from the Operational Server, or at least minimizing the time it spends there.

The best possible score is **0** (attacker never compromises anything, no Restore needed). Every step the score goes down.

**r_step = r_exploit + r_restore + r_impact**

|Source|Trigger|Penalty|
|---|---|---|
|r_exploit|Each User Host or Op Host currently compromised|−0.1 per host|
|r_exploit|Each Enterprise Host or Op Server currently compromised|−1.0 per host|
|r_restore|Blue agent uses Restore on any host|−1.0 flat|
|r_impact|Red agent is actively Impacting the Op Server|−10.0 per step (persists until restored)|
The **episode return R = Σ r_step** over 30 steps, so the worst theoretical score is around -218 (sleeping agent, attacker impacts the Op Server nearly every step).
## 5.3. Notable Results

| Finding                            | Detail                                                                                                                                                                          |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Overall improvement**            | FORGE achieves 1.7–7.7× improvement over zero-shot and 29–72% over Reflexion across all 12 model–representation conditions                                                      |
| **Best absolute return**           | Gemini + Examples: mean −24.5 (vs. zero-shot −189.6); peak checkpoint −3.60 approaching the DRL top score of −3.47                                                              |
| **Catastrophic failure reduction** | Major-failure rate (return < −100) drops from 90% (zero-shot) to ~1% under strongest FORGE configurations                                                                       |
| **Best representation**            | Examples achieves the highest return for 3/4 models; Rules achieves 40% fewer tokens with comparable final performance                                                          |
| **Broadcast is essential**         | No-graduation variant (broadcast only, no freezing) also outperforms Reflexion in all 12 conditions, confirming broadcast — not graduation — carries the performance gains      |
| **Graduation saves compute**       | Removing graduation roughly doubles adaptation token cost per instance by Stage 6, while improving final return for Grok and Qwen but hurting Gemini and Llama                  |
| **Weaker models benefit more**     | Improvement inversely correlates with baseline strength: Gemini (worst baseline −189.6) gains 7.7×; Grok (best baseline −58.4) gains 1.7×                                       |
| **Failure trigger sensitivity**    | τ = −11.0 (only severe failures trigger reflection) outperforms the submitted τ = −1.1 for Gemini Rules (mean −24.6 vs. −30.6), suggesting cleaner signal at harsher thresholds |
| **Token cost (Gemini)**            | Rules: 106M total tokens; Examples: 177M; Mixed: 188M                                                                                                                           |

---

# 6. Strengths

- **Population broadcast as a structural fix to Reflexion's instability.** The key insight that the bottleneck in prompt-only adaptation is not reflection quality but the absence of selection pressure is both well-motivated and empirically confirmed by the ablation. 

- Rules, Examples, and Mixed are compared under identical training dynamics, providing the first controlled study of artifact type efficiency in an adversarial POMDP 


---

# 7. Gaps

- **Single-best broadcast brittleness.** The champion selection is winner-take-all and uses a single frozen checkpoint episode, making it susceptible to lucky high-variance outcomes. 
	=> Alternative aggregation strategies (e.g., top-k averaging, multi-episode checkpointing, or uncertainty-aware selection) or artifacts merging

- Only test in one niche domain

- **Graduation is model-dependent and not well-understood:** Removing graduation helps Grok and Qwen but hurts Gemini and Llama, with no clear mechanistic explanation.

- Did not compare to other system such as ALMA, ACE, Dynamic Cheatsheet,...

---

# 8. Highlights

> [!PDF|255, 208, 0] [[FORGE.pdf#page=1&annotation=655R|FORGE, p.1]]
> > FORGE (FailureOptimized Reflective Graduation and Evolution), a staged, populationbased protocol that evolves prompt-injected natural-language memory for hierarchical ReAct agents.

> [!PDF|255, 208, 0] [[FORGE.pdf#page=1&annotation=664R|FORGE, p.1]]
> >  a staged population protocol where 𝑁 hierarchical ReAct agents evolve prompt-injected memory over 𝑆 stages




