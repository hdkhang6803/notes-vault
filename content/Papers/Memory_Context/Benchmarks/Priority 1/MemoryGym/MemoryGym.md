---
Tags:
Date: "2024"
Authors: Yiming Xiong, Shengran Hu, Jeff Clune (University of British Columbia / Vector Institute)
Venue: JMLR
Paper: "Memory Gym: Towards Endless Tasks to Benchmark Memory Capabilities of Agents"
---
## 1. Terminology

| Term                                        | Definition                                                                                                                                                                                                                                                                                                                                     |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Memory Gym                                  | A benchmark suite of three 2D partially-observable DRL environments (Mortar Mayhem, Mystery Path, Searing Spotlights), each with a finite and an endless variant, exposing 84×84×3 RGB visual observations and multi-discrete action spaces.                                                                                                   |
| Mortar Mayhem (MM)                          | A grid-arena environment in which the agent must memorize a sequence of movement commands and then execute each in the observed order, earning +0.1 reward per correct execution.                                                                                                                                                              |
| Mystery Path (MP)                           | An environment requiring the agent to traverse an invisible, procedurally generated path from a randomly sampled cardinal origin, being reset to the origin whenever it steps off the path.                                                                                                                                                    |
| Searing Spotlights (SS)                     | A pitch-black arena illuminated only by moving spotlights, in which the agent starts with a fixed health pool, loses health when caught by a spotlight, and must track its own hidden location from memory of past actions/positions to collect coins and (in the finite variant) reach an exit.                                               |
| Endless task                                | A task variant with no fixed terminal goal state; instead of ending at task completion, the episode's information-retention demand grows without bound as long as the agent keeps succeeding, terminating only on agent failure.                                                                                                               |
| Automatic curriculum                        | A difficulty-scaling mechanism in which task demands increase as a direct function of the agent's own success (e.g., each new correctly executed Mortar Mayhem command unlocks a longer command list), removing the need for a hand-scheduled curriculum.                                                                                      |
| Transformer-XL (TrXL)                       | A transformer variant (Dai et al., 2019) adapted here to a sequence-to-one PPO encoder that uses segment-level recurrence: cached per-timestep hidden states are stored in an episodic memory and retrieved via a fixed-length sliding attention window, with maximum context length L_max = N × (L − 1) + 1 for N layers and window length L. |
| Gated Recurrent Unit (GRU)                  | A recurrent neural network cell (Cho et al., 2014) used as the paper's non-transformer memory baseline, with a default hidden-state size of 512 (384 in one capacity-ablation).                                                                                                                                                                |
| Proximal Policy Optimization (PPO)          | The on-policy actor-critic RL algorithm (Schulman et al., 2017) underlying both baselines, optimized via the clipped surrogate objective $L^C_t(θ) = −E_t[min(q_t(θ)A_π^{GAE}, clip(q_t(θ), 1−ε, 1+ε)A_π^{GAE})]$.                                                                                                                             |
| Generalized Advantage Estimation (GAE)      | The advantage-estimation technique (Schulman et al., 2016) supplying $A_π^{GAE}(o_t, h_t, a_t)$ to the PPO objective.                                                                                                                                                                                                                          |
| Episodic Memory (TrXL)                      | A per-layer cache $L_l$ storing every past timestep's TrXL-layer input; at timestep t, keys/values are sliced from indices max(0, t − window length) to t − 1, with no gradient flow into the cache.                                                                                                                                           |
| Sliding Memory Window                       | The fixed-length window (length L) used to bound how far back TrXL's attention can retrieve cached hidden states, decoupling attendable history from raw sequence length.                                                                                                                                                                      |
| Observation Reconstruction Loss (Obs. Rec.) | An auxiliary head reconstructing the visual observation from the memory encoder's latent output via a transposed Atari CNN, optimized with binary cross-entropy $L^R_t = −[ô_t log(o_t) + (1 − ô_t)log(1 − o_t)]$.                                                                                                                             |
| Ground Truth Estimation head (GT)           | A diagnostic auxiliary head predicting the environment's next-target-position label, optimized with squared-error loss $L^Y_t = (ŷ_t − y_t)²$, used to test whether TrXL's capacity is the bottleneck rather than its learning signal.                                                                                                         |
| CleanRL                                     | The open-source single-file DRL implementation library (Huang et al., 2022b) into which the paper contributes its TrXL+PPO baseline.                                                                                                                                                                                                           |
| Gated Transformer-XL (GTrXL)                | A TrXL variant (Parisotto et al., 2020) adding a GRU-style gating mechanism, originally paired with V-MPO; tested here (untuned, bias 0 and bias 2) as a secondary finite-environment baseline.                                                                                                                                                |
| Task Progress                               | A normalized [0, 1] metric of finite-task completion fraction (e.g., commands executed ÷ 10 for MM) used for cross-environment aggregation.                                                                                                                                                                                                    |

---

## 2. Metadata

|Attribute|Value|
|---|---|
|Environments|3 base tasks (Mortar Mayhem, Mystery Path, Searing Spotlights) × 2 variants (finite, endless) = 6 primary tasks, plus 3 auxiliary "Grid"/"Act Grid" variants used only for minor-baseline ablations|
|Observation format|Visual: 84×84×3 RGB pixels (encoded via Atari CNN); optional vector/game-state observation (via FC encoder) for auxiliary baselines|
|Action format|Multi-discrete|
|Level generation|Procedural content generation; a new level is sampled from a distinct seed set each episode reset|
|Seeds per run|5 independent training seeds per configuration; 15-seed pooled aggregation for normalized cross-environment plots|
|Evaluation protocol|Each plotted data point = mean of 50 episode seeds × 3 stochastic-policy repetitions (finite envs)|
|Compute budget|25,000 GPU hours total (NVIDIA A100, 40GB VRAM)|
|Simulation speed|5,789–14,033 steps/sec depending on environment (Appendix F, Table 7)|
|Positional-encoding range|512 (finite envs) / 2048 (endless envs)|
|Training length (finite)|~500M steps (MM), ~400M steps (MP, SS)|
|Training length (endless)|~800M steps (all three)|
|License|CC-BY 4.0|
|Code / data release|[https://marcometer.github.io/jmlr_2024.github.io/](https://marcometer.github.io/jmlr_2024.github.io/) (also integrated into CleanRL)|

---

## 3. What it Measures (What)

### 3.1 Task Definition

- **Mortar Mayhem (finite):** the agent is immobile while observing a sequence of 10 movement commands, then must execute each in order on a grid arena; a wrong execution ends the episode, a correct one yields +0.1.
- **Mortar Mayhem (endless):** observation and execution alternate one command at a time — the list is never fully re-shown, only extended, so it can grow indefinitely as long as the agent keeps succeeding.
- **Mystery Path (finite):** the agent navigates an invisible path from a randomly sampled cardinal origin toward a goal; falling off resets it to the origin. Evaluation paths span 7–14 tiles.
- **Mystery Path (endless):** the path is generated indefinitely left-to-right (leftward movement is disabled); the agent gets a 20-step budget per tile, and the episode ends if it falls off before its best progress or falls at the same spot twice.
- **Searing Spotlights (finite):** in a dark arena lit only by moving spotlights, the agent (10 health points, −1 per hit) must collect one coin (+0.25) then reach an exit (+1), relying on memory of its own hidden position.
- **Searing Spotlights (endless):** the exit is removed; a new coin spawns after each collection (visible 6 frames, 160-step window), and spotlights spawn at a constant rate instead of intensifying.

### 3.2 Example Records

**Endless Mortar Mayhem** (Figure 1b growth pattern):

> Cycle 1 — observe `[↖]` → execute `[↖]` 
> Cycle 2 — observe `[→]` only (previous command not re-shown) → execute `[↖, →]` 
> Cycle 3 — observe `[→]` only → execute `[↖, →, →]` 
> Cycle 4 — observe `[↓]` only → execute `[↖, →, →, ↓]` … continues as long as every execution is correct.

**Endless Mystery Path:** the agent earns +0.1 for each newly reached, previously-unvisited tile; if it falls off at step _k_, all subsequent progress must be re-earned by retracing steps back past tile _k_, so reward density falls the longer recovery takes.

**Endless Searing Spotlights:** episode-level record consists of health (starts at 10, −1 per spotlight hit), a rolling coin (visible for 6 frames, 160 steps to collect before it's replaced), and continuous constant-rate spotlight spawning — episode ends only when health reaches 0.

### 3.3 Metrics

|Metric|Environment(s)|Definition|
|---|---|---|
|Task Progress (0–1)|All finite environments|Fraction of task completed (commands executed ÷ 10 for MM; tiles reached ÷ path length for MP; 50% coin + 50% exit for SS)|
|Success Rate|Finite MP variants; SS ablation study (Table 1)|Binary episode success/failure, averaged|
|Number of Commands Executed|Endless MM|Count of correctly executed commands before termination|
|Tiles Visited|Endless MP|Count of distinct, previously unvisited path tiles reached|
|Number of Coins Collected|Endless SS|Count of coins collected before health reaches 0|
|Normalized Score (0–1)|Cross-environment aggregation plots (Fig. 6d, 7d)|Per-environment raw score divided by the max mean GRU score in that environment, then averaged across the 15 pooled seeds|
|Episode Length (steps)|Supplementary comparison (§5.5)|Steps survived/completed per episode, used to contrast finite vs. endless memory demands (e.g., 274 steps to finish finite MM vs. 3,000+ for GRU in endless MM)|

---

## 4. Uniqueness (Why)

### 4.1 Other Benchmarks

|Benchmark|Modality|Note (per paper)|
|---|---|---|
|DeepMind Lab 30|First-person 3D navigation|Procedurally generated, but skyboxes are exploitable, reducing genuine memory demand|
|Minigrid|2D grid|T-Maze-inspired: memorize a goal cue, traverse an alley, choose the correct exit|
|Miniworld|First-person 3D|Task structure similar to Minigrid|
|VizDoom|First-person 3D shooter|Navigation plus computer-controlled enemies|
|Memory Task Suite|Mixed (PsychLab, Spot-the-Difference, navigation, ordering)|4 task categories spanning image recall, cue memorization, Morris-water-maze-style navigation, and transitive ordering|
|Procgen|2D|6 of 16 environments modified to be partially observable via a reduced/centered view|
|Numpad|Abstract sequence|Fixed-length key sequence memorized by trial-and-error (sequence never shown)|
|Memory Maze (1)|3D, Morris-water-maze-like|Agent must relocate an apple after being randomly repositioned|
|Ballet|Visual sequence|Up to 8 dances shown; agent must later identify one specific dance|
|PopGym|Vector-based, 15 environments|Diagnostic / control / noisy / game / navigation categories, tuned for rapid convergence|
|Memory Maze (2)|3D procedurally generated mazes|First-person object-finding; also releases an offline-RL dataset|

### 4.2 Uniqueness

- Every benchmark above is **strictly finite** (episodes end at a bounded terminal state regardless of outcome) so cross-method comparisons default to _sample efficiency_ (steps to solve a fixed task) rather than _memory effectiveness_ (how much information can be retained, and for how long). 
- Memory Gym's endless variants are the only DRL memory benchmark **offering open-ended tasks** with a built-in automatic task difficulty scaling indefinitely with the agent's own competence rather than a fixed budget.
- It also **runs substantially faster** than most comparable benchmarks (5,789–14,033 steps/sec; only Procgen is faster) 
- unlike closed-source baselines such as MRA, HCAM, and GTrXL, ships an **openly reproducible TrXL+PPO reference implementation** in CleanRL.

---

## 5. Generation Method (How)

Using **Endless Mortar Mayhem** as the running example, task instances are generated and scaled as follows:

1. **Episode initialization:** a fresh arena is procedurally sampled from a distinct seed for each episode reset (5×5 grid by default).
2. **Command pool:** a discrete set of movement commands is defined (9 available commands in the endless variant, e.g. move to an adjacent tile in one of several directions, or remain in place).
3. **Curriculum step (observe → execute, alternating):**
    
    > Episode begins: 
    > - command 1 is displayed → agent executes command 1. 
    > - Next cycle: only command 2 is displayed (command 1 is _not_ re-shown) → agent must now execute commands 1 and 2, in order. 
    > - Next cycle: only command 3 is displayed → agent executes commands 1–3, in order. 
    > - …and so on, with the to-be-executed list growing by exactly one command per successful cycle.
    
4. **Reward assignment:** +0.1 per correctly executed command in sequence; no explicit failure penalty (Reward Command Failure = 0).
5. **Termination:** the episode ends the first time the agent executes a command incorrectly (this is the mechanism that makes the "endless" curriculum self-limiting in practice even though it is unbounded in principle).

- Endless Mystery Path streams path tiles indefinitely left-to-right with a 20-step time budget per tile,
- Endless Searing Spotlights spawns a fresh coin (6-frame visibility, 160-step collection window) each time the previous one is collected, with spotlights spawning at a constant rate rather than intensifying.

Separately, to make the benchmark usable, the paper also contributes an architecture that couples Atari-CNN/FC observation encoders to either a GRU or a TrXL memory encoder, trained with PPO's clipped objective plus optional observation-reconstruction and ground-truth-estimation auxiliary losses

---

## 6. Gaps

- The root cause of TrXL's endless-task underperformance is not fully resolved; five hypotheses 
	- inadequate network capacity
	- weak learning signal
	- initial-query temporal information
	- off-policy staleness in the episodic memory
	- positional-encoding indistinguishability
- Significant related baselines (MRA, HCAM, GTrXL as originally published) are excluded from the primary comparison due to closed-source unavailability, limiting external validity.
- Hyperparameter tuning was concentrated on the finite environments with a deliberately restricted search space (no full permutation sweep); the authors state they "do not claim to provide optimal hyperparameters."
- GTrXL and LSTM results (Appendix D) are reported without tuning, and the authors explicitly caution against drawing strong conclusions from them.
- Five seeds per configuration is acknowledged as potentially "insufficient for drawing strong conclusions" at the level of individual per-run curves (mitigated only by the 15-seed pooled aggregation).
- Wall-time comparisons are cluster-dependent and not perfectly controlled (Endless Mortar Mayhem times were measured on Noctua2; Mystery Path times on LiDo3, different CPUs/RAM).
- Endless-task episode length is itself agent-competence-dependent (better agents survive longer, incurring more simulation/inference cost per fixed step budget), which could subtly bias fixed-step-budget comparisons in favor of weaker, faster-terminating agents on wall-time metrics.
- Only two memory architecture families (GRU, TrXL) receive full tuning and evaluation; more recent mechanisms (e.g., structured state-space models, mentioned only as future work) are untested, so the benchmark's core empirical claim — that endless tasks reveal effectiveness differences finite tasks miss — is so far demonstrated for only this one architecture pairing.

---

## 7. Highlights