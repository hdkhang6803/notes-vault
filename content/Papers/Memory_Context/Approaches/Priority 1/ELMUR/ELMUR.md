---
Date: February 10, 2026
Authors: Yiming Xiong, Shengran Hu, Jeff Clune (University of British Columbia / Vector Institute)
Venue:
Paper:
Memory type:
  - Token-level
Agent env: Multi-agent
Record format: Text
Memory architecture:
  - 3-tier
  - graph-a
Tackle Module: Memory design
Need offline initialization: false
Fine-tuning?: false
Other tags:
---
## 1. Terminology

| Term                     | Definition                                                                                                                                                                                                            |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ELMUR                    | External Layer Memory with Update/Rewrite: a GPT-style transformer decoder where every layer carries its own external memory track alongside the token track                                                          |
| Token track              | The classic transformer path: self-attention over tokens within a segment, causal mask                                                                                                                                |
| Memory track             | Parallel path holding per-layer memory embeddings that persist across segments                                                                                                                                        |
| mem2tok                  | Cross-attention block where tokens (Q) read from memory (K, V), the "read" path                                                                                                                                       |
| tok2mem                  | Cross-attention block where memory (Q) reads from token hidden states (K, V) — the "write" path                                                                                                                       |
| LRU memory module        | Least-Recently-Used update rule that fills empty memory slots by replacement, then refreshes the oldest slot via convex blending                                                                                      |
| Segment-level recurrence | Trajectory of length T split into S segments; memory is detached and carried forward between segments, treating the transformer as an RNN over segments                                                               |
| Anchor (p)               | Per-slot timestamp recording the last time that memory slot was written                                                                                                                                               |
| Relative bias (B_rel)    | Learned per-head embedding indexed by clamped offset between token position and memory anchor, added to cross-attention logits                                                                                        |
| Convex blending (λ)      | Update rule $m' = λ·m_new + (1−λ)·m_old$ applied once all memory slots are full                                                                                                                                       |
| ObsEncoder               | Encodes raw observations into token embeddings before the first layer                                                                                                                                                 |
| ActionHead               | Maps final-layer hidden states to an action distribution (policy component)                                                                                                                                           |
| DeepSeek-MoE FFN         | Mixture-of-Experts feed-forward block (routed + shared experts, top-k routing) used in place of a standard MLP in both token and memory tracks                                                                        |
| Effective horizon H(ε)   | Number of environment steps a stored contribution remains influential before decaying below threshold ε; H(ε) = M·L·ln(ε)/ln(1−λ)                                                                                     |
| POMDP                    | Partially Observable Markov Decision Process, (S, A, O, T, Z, R, ρ₀, γ)                                                                                                                                               |
| Segments                 | a chunk of the trajectory used for segment-level recurrence. Given a context length L (the model's actual attention window) and a full trajectory of length T, the trajectory is partitioned into: $S=⌈T/L⌉$ segments |

## 2. Summary

ELMUR augments every layer of a transformer decoder with a bounded, persistent external memory that reads from and writes to token representations via bidirectional cross-attention (mem2tok / tok2mem), managed by a Least-Recently-Used (LRU) rule that fills empty slots first and then refreshes the oldest slot through convex blending. Combined with segment-level recurrence and a relative temporal bias, this lets a model trained with a short attention window (e.g., L=10) retain task-relevant cues across corridors up to one million steps, roughly 100,000× its native context. The authors back this with a theoretical analysis showing exponential forgetting and bounded memory norms under the LRU rule, and validate it empirically on T-Maze, MIKASA-Robo manipulation, and 48 POPGym tasks, where ELMUR matches or beats transformer, state-space, and offline-RL baselines while adding little compute overhead.

## 3. Why

Long-horizon, partially-observable decision making runs into two problem classes that ELMUR targets directly:
- **fixed attention-window limitations**: standard transformers truncate history, so information needed for a decision is lost once it falls outside the context window, and naively extending context is quadratically expensive — VLA and IL policies built on fixed-window transformers inherit this truncation-induced forgetting.
- **sample-inefficient long-horizon credit assignment**: sparse and delayed rewards in robotic manipulation make it costly to learn which distant past event mattered for a current decision, and reshaping rewards to compensate requires domain knowledge and risks bias. 
ELMUR's central bet is that persistent, layer-local, bounded-capacity memory can solve both without abandoning the transformer's local modeling strengths.

## 4. How

### 4.1 Memory Architecture — what does ELMUR's external memory actually look like?
![[ELMUR.pdf#page=2&rect=108,593,505,745&color=yellow|ELMUR, p.2]]
![[ELMUR.pdf#page=3&rect=306,367,505,492&color=yellow|ELMUR, p.3]]

Every one of ELMUR's N layers carries its **own** external memory track that runs in parallel with the standard token track, rather than a single memory shared across the network (Figure 1). Concretely, each layer ℓ maintains:

- **M memory slots**, $m ∈ R^{M×d}$, one vector per slot at the model dimension d
- **Anchors**, $p ∈ Z^M$, one integer timestamp per slot recording when that slot was last written
- A **token track**, processing the current segment's observations into actions via causal self-attention
- A **memory track**, holding the M slots above, persisting across segments via segment-level recurrence (detached, carried forward as an RNN would carry hidden state)

> **Running example (Figure 2).** At initialization, a layer's M slots are filled with random vectors ~N(0, σ²I) and every anchor is set to a sentinel value of −1, marking all slots empty. This is the blank memory a fresh trajectory starts from, before any segment has been processed.

> [!design-rationale] Giving each layer its own memory (rather than one memory shared network-wide) is a deliberate layer-local design choice
>  the ablations later confirm that a shared-memory variant degrades performance substantially (score drops from 1.00 to 0.45), since different layers evidently need to retain different kinds of information.

The two tracks are coupled only through cross-attention: tokens read from memory via **mem2tok**, and memory is written from tokens via **tok2mem**
### 4.2 Retrieve Flow (mem2tok) — how do tokens pull information out of memory?

Self-attention with relative positional encoding and a causal mask first handles local, within-segment dependencies:

hsa=AddNorm(x+SelfAttention(x))h_{sa} = \text{AddNorm}(x + \text{SelfAttention}(x))hsa​=AddNorm(x+SelfAttention(x))

Long-range context then comes from memory instead of a longer window — tokens query memory via mem2tok, with memory embeddings acting as keys and values:

hmem2tok=AddNorm(hsa+CrossAttention(Q=hsa,K=m,V=m))h_{mem2tok} = \text{AddNorm}(h_{sa} + \text{CrossAttention}(Q=h_{sa}, K=m, V=m))hmem2tok​=AddNorm(hsa​+CrossAttention(Q=hsa​,K=m,V=m))

followed by a DeepSeek-MoE feed-forward block:

h=AddNorm(hmem2tok+FFN(hmem2tok))h = \text{AddNorm}(h_{mem2tok} + \text{FFN}(h_{mem2tok}))h=AddNorm(hmem2tok​+FFN(hmem2tok​))

> **Running example continued.** By segment M+2, several of the layer's slots hold blended content from earlier segments (see §4.4). A token processed in segment M+2 queries all M slots through mem2tok — including the long-stale slot 1 from the very first segment — allowing an early cue to influence a decision made far outside the model's L-token attention window.

> [!design-rationale] The authors swap the usual MLP-based FFN (as in Decision Transformer) for a DeepSeek-MoE FFN, following DeepSeek-V3's design, to gain parameter efficiency and specialization without proportional compute cost — this is what keeps ELMUR's per-step latency below RATE and DT despite its added memory machinery.

### 4.3 Update/Write Flow (tok2mem + LRU) — how does memory get refreshed without simply overwriting itself?

After a segment is processed, token states update memory through tok2mem (same cross-attention mechanism, roles reversed, non-causal mask, reversed relative bias):

mtok2mem=AddNorm(m+CrossAttention(Q=m,K=h′,V=h′))m_{tok2mem} = \text{AddNorm}(m + \text{CrossAttention}(Q=m, K=h', V=h'))mtok2mem​=AddNorm(m+CrossAttention(Q=m,K=h′,V=h′)) mnew=AddNorm(mtok2mem+FFN(mtok2mem))m_{new} = \text{AddNorm}(m_{tok2mem} + \text{FFN}(m_{tok2mem}))mnew​=AddNorm(mtok2mem​+FFN(mtok2mem​))

`m_new` is a _candidate_ update — it is not written directly into memory. Instead it is merged into existing slots via the LRU rule: while empty slots remain, `m_new` is written by full replacement; once all slots are full, the least-recently-used slot (smallest anchor) is refreshed by convex blend instead of being overwritten outright:

mji+1=λ mnewi+1+(1−λ) mjim^{i+1}_j = \lambda\, m^{i+1}_{new} + (1-\lambda)\, m^i_jmji+1​=λmnewi+1​+(1−λ)mji​

> **Running example continued.** Segment 1 writes into empty slot 1 by full replacement (anchor set to K−1). Segment 2 fills slot 2 (anchor 2K−1), and so on until segment M fills the last slot. From segment M+1 onward all slots are full, so the least-recently-used slot — slot 1, with the smallest anchor — is refreshed via convex blending: `m^{[M+1]}_1 = λ·m_new + (1−λ)·m^{[1]}_1`, combining the new segment's content with what was already there rather than discarding it.

> [!design-rationale] Filling empty slots first before blending lets the model use its full memory budget before any information is diluted — early segments are stored losslessly, and only once capacity is exhausted does the model trade stability against plasticity via λ.

### 4.4 Relative Bias — resolving ambiguous absolute positions across segments

> **Running example continued.** By segment M+2, slot 1 already holds a blend from segment M+1 with anchor (M+1)K−1. A token in the current segment sitting at absolute position t needs to know _how long ago_ that slot was last touched, not just _where_ it sits in absolute terms — since positions repeat every segment.

The bias is derived from clamped pairwise offsets between token position t and memory anchor p, indexed into a learnable per-head embedding table E:

Attn(Q,K)=QK⊤dh+Brel,Brel={E[t−p]mem2tok (read)E[p−t]tok2mem (write)\text{Attn}(Q,K) = \frac{QK^\top}{\sqrt{d_h}} + B_{rel}, \quad B_{rel} = \begin{cases} E[t-p] & \text{mem2tok (read)} \\ E[p-t] & \text{tok2mem (write)} \end{cases}Attn(Q,K)=dh​​QK⊤​+Brel​,Brel​={E[t−p]E[p−t]​mem2tok (read)tok2mem (write)​

> [!design-rationale] Using _relative_ rather than absolute time keeps the read/write bias meaningful even though the same token position can correspond to wildly different points in a trajectory once memory persists across many segments — the read and write paths share one embedding table but can still learn distinct temporal preferences.

### 4.5 Theoretical Guarantees on the LRU Rule — is bounded capacity actually safe?

Algorithm 2 (§4.3) is backed by formal analysis in the paper:

- _Exponential forgetting_ — after k overwrites, the original content's coefficient is (1−λ)^k, and a write performed τ updates ago contributes λ(1−λ)^{τ−1}; half-life k₀.₅ ≈ ln2/λ as λ→0.
- _Effective horizon_ — H(ε) = M·L·ln(ε)/ln(1−λ), scaling linearly with both memory size M and segment length L.
- _Memory boundedness_ — if every write and the initial memory are norm-bounded by C, every memory embedding stays within the closed ball of radius C for all time, by induction on the convex-combination structure of the update.

## 5. Benchmarks

### 5.1 Baselines

|Baseline|Category|
|---|---|
|Decision Transformer (DT)|Transformer, offline sequence modeling|
|RATE|Transformer, memory-augmented (concatenates memory with tokens)|
|RMT|Recurrent Memory Transformer|
|TrXL|Transformer-XL style context extension|
|DMamba (Decision Mamba)|State-space model, efficient recurrence|
|BC-MLP|Behavior Cloning, MLP policy|
|BC-LSTM|Behavior Cloning, recurrent policy|
|CQL(-MLP)|Conservative Q-Learning, offline RL|
|DP|Diffusion Policy|
|Random / Persistent agent|Lower-bound reference policies (T-Maze only)|

### 5.2 Benchmarks used

- [[T-Maze]] — synthetic corridor task testing single-cue retention across corridors up to one million steps
- [[MIKASA-Robo]] — sparse-reward robotic manipulation suite with RGB visual observations (RememberColor3/5/9-v0, TakeItBack-v0)
- [[POPGym]] — 48-task suite of partially observable puzzles and control environments

### 5.3 Results by task type

**Synthetic (T-Maze):** 100% success rate up to corridor lengths of 1,000,000 steps with a context length of only L=10 (S=3 segments) — a retention horizon ~100,000× the attention window. Generalization across 11 train/test length pairs (9–9600 steps) is perfect in both interpolation and extrapolation directions.

**Robotic manipulation (MIKASA-Robo, Table 1):** ELMUR leads on all four tasks — RememberColor3-v0 0.89±0.07 (next best RATE 0.65±0.04), RememberColor5-v0 0.19±0.03, RememberColor9-v0 0.23±0.02, TakeItBack-v0 0.78±0.03 (next best RATE 0.42±0.24) — with stable performance as distractor count increases.

**Puzzle/control (POPGym-48, Table 2):** Best aggregate score across all 48 tasks (10.4 vs. RATE 9.5, BC-LSTM 9.0, DT 5.8), driven mainly by memory-intensive puzzles (1.2 vs. 0.45 for RATE; DT and BC-LSTM negative), while remaining competitive on reactive/control tasks (9.2, close to DT's 9.3). Ranks first on 24/48 tasks overall.

## 6. Strengths

- Retention horizon scales with memory size M and segment length L rather than sequence length, giving predictable, bounded compute cost even for million-step corridors.
- Backed by formal guarantees (exponential forgetting, bounded memory norm) rather than purely empirical claims about long-horizon retention.
- Consistent gains hold across three structurally different benchmark types (synthetic, visual robotic manipulation, puzzle/control), not just one narrow setting.
- Comes with essentially no efficiency penalty — 2.1M parameters and 6.8ms/step, faster than RATE (7.2ms) and DT (10.7ms) despite the added memory machinery.

## 7. Gaps

**Fixed, non-adaptive blending factor.** λ is a single tunable hyperparameter shared across all slots and the whole trajectory, and the ablations show intermediate λ (≈0.4–0.6) is unstable while under-provisioned memory (M<N) is highly sensitive to λ, σ, and segmentation. => Explore adaptive or per-slot blending rules that adjust plasticity based on content salience rather than a fixed global λ.

**Evaluation confined to simulation and imitation learning.** The paper explicitly omits real-robot experiments (to avoid latency/reset/safety confounds) and online RL baselines (incomparable training budgets), so all results are IL-from-demonstrations in simulated environments. => Extend evaluation to online RL and real-robot deployment, as the authors themselves flag in their limitations.

**Memory capacity must be manually matched to task structure.** Performance is near-perfect when M ≥ N (number of segments needed) but drops sharply when M < N, meaning practitioners must know or estimate the required number of segments in advance. => Investigate mechanisms for dynamically growing or reallocating memory slots when task-required capacity is unknown ahead of time.

## 8. Highlights

> [!PDF|255, 208, 0] [[ELMUR.pdf#page=11&annotation=985R|ELMUR, p.11]]
> > Transformer-xl: Attentive language models beyond a fixed-length context.
> 
> 