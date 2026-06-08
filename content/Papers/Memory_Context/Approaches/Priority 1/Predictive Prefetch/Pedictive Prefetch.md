---
Tags:
Date: "2026"
Authors: Wuyang Zhang, Shichao Pei
Venue: ICML
Paper: Predictive Prefetching for Retrieval-Augmented Generation
---
# 1. Terminology

| **Term**                   | Definition                                                                                                                                                                                                                                                                                        |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Predictive Prefetching** | Initiating retrieval _before_ uncertainty materializes, based on learned precursor signals in generation dynamics                                                                                                                                                                                 |
| **Synchronous RAG**        | Standard RAG where token generation fully blocks while awaiting retrieval results                                                                                                                                                                                                                 |
| **Asynchronous RAG**       | Retrieval executes concurrently with generation; generation does not pause                                                                                                                                                                                                                        |
| **Retrieval Predictor**    | Lightweight 2-layer transformer encoder that estimates probability of impending retrieval need within a lookahead horizon Δ                                                                                                                                                                       |
| **Context Monitor**        | Component that determines the optimal number of tokens to wait (k ∈ {0,...,5}) before issuing a retrieval query, via `ContextScore`, `SufficiencyClassifier`, and `ClarityScore`                                                                                                                  |
| **Query Generator**        | Fine-tuned T5-small model that generates a context-aware retrieval query from accumulated context after the waiting period                                                                                                                                                                        |
| **Entropy Threshold θ**    | Scalar threshold on token-level generation entropy $H(p_t)$ that defines a "high uncertainty" retrieval trigger point                                                                                                                                                                             |
| **Lookahead Horizon Δ**    | The number of future tokens within which the predictor forecasts an entropy threshold crossing (set to 10 tokens)                                                                                                                                                                                 |
| **Lead Time**              | Tokens of advance notice provided by the predictor before actual uncertainty, determines how much time is available for retrieval to complete asynchronously                                                                                                                                      |
| **TTFT**                   | Time-to-First-Token: latency from query submission to first generated token                                                                                                                                                                                                                       |
| **E2E Latency**            | End-to-end generation latency including all retrieval operations                                                                                                                                                                                                                                  |
| **Efficiency Score**       | F1 × 1000 / E2E (ms); composite metric trading answer quality against generation speed                                                                                                                                                                                                            |
| **QAL**                    | E2E / (EM/100); Quality-Adjusted Latency, lower is better                                                                                                                                                                                                                                         |
| **QRS**                    | Query Relevance Score: cosine similarity between query embedding and retrieved document embeddings                                                                                                                                                                                                |
| **Semantic Precursor**     | Characteristic patterns in entropy trajectories (derivative of token entropy of the final output), attention allocation (entropy of Q*K), and value representation dynamics (norm of change in value vector of attention mechanism) that emerge 8–16 tokens _before_ uncertainty becomes critical |
| **REUSE**                  | Policy action: cached documents satisfy current information need; skip new retrieval                                                                                                                                                                                                              |
| **ACCUMULATE**             | Policy action: context is not yet syntactically complete; defer query construction by 1–2 extra tokens                                                                                                                                                                                            |
| **FETCH**                  | Policy action: issue asynchronous retrieval                                                                                                                                                                                                                                                       |
| **GENERATE**               | Policy action: continue generation without retrieval                                                                                                                                                                                                                                              |
| **Contextual Bandit**      | RL formulation used here, each decision receives immediate, action-specific feedback; no long-horizon credit assignment needed                                                                                                                                                                    |
| **Contriever**             | A dense retrieval model developed by Meta that maps text to a fixed-dimensional vector (typically 768-d) such that semantically related passages end up close in embedding space, measured by cosine similarity.                                                                                  |

---

# 2. Paper Summary (What)

This paper proposes a framework for **predictive asynchronous retrieval** in RAG systems. 
The proposed system, through three jointly-trained components (Retrieval Predictor, Context Monitor, Query Generator), learns to detect uncertainty _precursors_ 8–16 tokens in advance and constructs semantically aligned queries before uncertainty peaks.
Retrieval runs concurrently in a prefetch thread pool; generation is never blocked. If a prediction fails, the system falls back to synchronous retrieval gracefully. 

---

# 3. What it Solves (Why)

- _Reactive adaptive RAG_ (FLARE, Self-RAG, DRAGIN): Detect uncertainty after it manifests, then block to retrieve. Latency is not hidden.
- _Heuristic asynchronous RAG_ (TeleRAG, PipeRAG): Overlap retrieval with generation, but use stale token as query or fixed-interval queries. Topic boundaries shift during multi-domain generation, causing retrieved content to misalign with actual needs, leading to hallucination or self-correction overhead.

---

# 4. Methodology (How)
![[Predictive Prefetch.pdf#page=4&rect=55,595,293,727|Predictive Prefetch, p.4]]
The framework decouples generation and retrieval into concurrent threads.
At each generation step, three lightweight components inspect internal LLM signals to make a cascade of decisions. 

>**Running example:** A multi-hop QA model is generating the answer to _"Who directed the film that starred the actor born in the same city as Marie Curie?"_
>
The model has already established that Marie Curie was born in Warsaw. At token T2, the generated context reads: _"The actor born in Warsaw is..."_

## 4.1. Retrieval Predictor: _When should retrieval be triggered?_

- At every step t, the predictor reads internal LLM representations from **middle-upper layers** (30–45% of model depth: layers 10–14 in a 32-layer Llama). 
> 	<span style="color:rgb(0, 112, 192)">This depth range is chosen because interpretability research shows intermediate layers capture high-level semantic abstractions while preserving uncertainty signals; final layers overfit to output distributions.</span>

- Three signal types are extracted:
	- **$H_t$**: hidden states over a 16-token sliding window (encodes semantic abstractions, entity types) of LLM's FFN
	- **$A_t$**: attention weight matrices (focus patterns; scattered attention signals approaching knowledge boundaries)
	- **$V_t$**: value vectors and their temporal changes (information flow; norm spikes when model shifts from recall to multi-step reasoning)
	- **$o_t$**: output distribution statistics (entropy $H(p_t)$, top-k probability margins, tail mass)

- These are concatenated and passed through a **2-layer transformer encoder**, whose output will be fed into a prediction head:
$$
z_t = TransformerEncoder([H_t; A_t; V_t])  ∈ ℝ^{512}
$$
$$
p̂_t = σ(W_p · [z_t; o_t] + b_p)
$$
	`p̂_t` estimates the probability that token-level entropy will exceed threshold θ within the next Δ = 10 tokens.
- If `p̂_t > τ_rag = 0.65`, the system proceeds to the Context Monitor.

> At T2, the predictor sees rising attention dispersion and increasing entropy derivative in the hidden states. 
> 
> It outputs p̂_T2 = 0.87, high confidence that retrieval will be needed around T5, when the model must commit to naming the actor. 
> 
> The current context _"The actor born in Warsaw is..."_ ends on an incomplete copula --> the model is about to generate a proper noun it may not know confidently.

---
## 4.2. Context Monitor: _Is the current context sufficient to support retrieval?_

>Triggering retrieval at T2 would generate a query from a fragment: _"The actor born in Warsaw is..."_ --> too incomplete to retrieve useful documents. The Context Monitor decides how long to wait.

**Phase 1: Context Accumulation:** A T5-based prediction head (ContextScore) scores expected query quality for each possible wait time k ∈ {0,...,5} based on current context:

$$
k^* = \underset{k∈{0,...,5}} {\operatorname{argmax}} ContextScore(c_{t+k})
$$
><span style="color:rgb(0, 112, 192)">The ContextScore outputs 6 scores for futures tokens by using the current context only, where the futures are not visible yet.</span>

> **Example:** 
> ContextScore predicts: 
> - k=0 → 0.55
> - k=1 → 0.61
> - k=2 → 0.74
> - k=3 → 0.81
> - k=4 → 0.79
> - k=5 → 0.76. 
> - The optimal wait is **k* = 3 tokens**. 
> - The system stores k*=3 in the **ContextBuffer** and generation continues uninterrupted.
> - After waiting, the accumulated context at T5 reads: _"The actor born in Warsaw is a Polish actor"_ --> a complete, unambiguous phrase matching what ContextScore anticipated

**Phase 2: Adaptive Query Construction:** After $k^*$ tokens are generated, two additional checks before issuing the query:
- **SufficiencyClassifier:** 
	- Computes cosine similarity between current context embedding and cached document embeddings.
	- If similarity > 0.8, the existing cache already covers the information need -><span style="color:rgb(192, 0, 0)"> skip retrieval. </span>

> **Example:** 
> - SufficiencyClassifier returns 0.31
> - The cache (containing Marie Curie background documents from an earlier retrieval) does not cover the actor or director. Retrieval proceeds.

- **ClarityScore:** 
	- Evaluates syntactic completeness. 
	- Score ≥ 0.7 means the context forms a complete phrase; lower scores trigger<span style="color:rgb(192, 0, 0)"> 1-2 extra tokens of waiting.</span>

> **Example:** ClarityScore = 0.84 at k*=3, confirming the phrase is syntactically complete. The system is cleared to generate a query.

---
## 4.3. Query Generator: _What should be retrieved?_
A fine-tuned **T5-small** (60M parameters) generates a retrieval query from the accumulated context:

$$
q = T5(c_{t+k^*})
$$

Prediction confidence modulates strategy:
- High confidence (`p̂ > 0.8`): focused, specific query
- Medium confidence (0.5–0.8): 2–3 diverse query variants
- Low confidence (≤ 0.5): broader exploratory retrieval

>With p̂_T2 = 0.87 (high confidence) and context _"The actor born in Warsaw is a Polish actor"_, T5 generates the focused query: **"famous Polish actor born Warsaw"**.
>This is dispatched asynchronously while generation continues past T5. 
>The document arrives at T6, aligned with actual needs when the model must commit to the actor's name.

---
## 4.4. Learning and Optimization
**Multi-task pretraining** uses oracle labels derived from paired generation runs (with and without retrieval) on HotpotQA and NQ (~50K position-level instances). Retrieval utility $s = EM_{with} − EM_{without}$ provides richer supervision than entropy alone. 
The joint loss:

$$
L = α·L_{pred} + β·L_{timing} + γ·L_{suff} + δ·L_{clarity} + ε·L_{query}
$$

with weights (1.0, 0.5, 0.5, 0.3, 1.0): prediction and query generation receive highest weight.

**Online adaptation** via policy gradient (REINFORCE) continues during deployment. 
- The policy $π_φ$ has four actions (GENERATE, REUSE, ACCUMULATE, FETCH) with action-specific rewards. 
- The contextual bandit structure yields immediate per-decision feedback:
$$ ∇ϕ​J=E_{s∼ρ​}[∇_ϕ​logπ_ϕ​(a∣s)⋅R(s,a)]$$
- The policy $\pi_\phi$​ observes a state, takes an action, receives a reward, and **updates its parameters** to make good actions more probable and bad actions less probable.

|Action|Condition|Reward|
|---|---|---|
|GENERATE|Retrieval not needed, correctly skipped|+0.3|
|GENERATE|Missed beneficial retrieval|−0.8|
|REUSE|Cache sufficient|+1.0|
|REUSE|Cache insufficient|−0.5|
|ACCUMULATE|Deferral improves query|+0.2|
|ACCUMULATE|Excessive delay (>5 tokens)|−0.3|
|FETCH|Retrieval improves answer|+1.0|
|FETCH|Unnecessary retrieval|−0.5|
|FETCH|Late arrival blocks generation|−2.0|


---

# 5. Benchmarks

## 5.1. Other Baselines

|Method|Timing|Execution|Query Signal|Key Limitation|
|---|---|---|---|---|
|No-RAG|—|—|—|No external knowledge|
|Sync-RAG|Reactive|Sync|Raw tokens|Fully blocking|
|Self-RAG|Reactive|Sync|Learned tokens|Requires LLM fine-tuning|
|Adaptive-RAG|Reactive|Sync|Learned (complexity)|Still blocking|
|DRAGIN|Reactive|Sync|Attention-weighted|Reactive, no lookahead|
|FLARE|Reactive|Sync|Low-conf. predictions|Regeneration overhead|
|Entropy-Threshold|Reactive|Sync|Entropy only|Simple heuristic|
|PipeRAG|Reactive|Async|Stale raw tokens|Query staleness|
|Oracle|Gold annotations|Sync|Perfect|Upper bound only|

## 5.2. Benchmarks

| Benchmark             | Type            | Retrieval Latency | Notes                                  |
| --------------------- | --------------- | ----------------- | -------------------------------------- |
| [[HotpotQA]]          | Multi-hop QA    | 125 ms (FAISS)    | Primary; bridge + comparison questions |
| [[2WikiMultiHopQA]]   | Multi-hop QA    | 125 ms            | Controlled reasoning types             |
| [[Natural Questions]] | Single-hop QA   | 125 ms            | Real search queries                    |
| [[TriviaQA]]          | Factual QA      | 125 ms            | Trivia with ~6 evidence docs each      |
| [[RepoBench-P]]       | Code completion | 125 ms            | Cross-file dependency retrieval        |
| [[QMSum]]             | Summarization   | 50–100 ms         | Local vector DB; lower-latency regime  |

## 5.3. Notable Results

**QA (Llama-3.1-8B, Table 2):**

| Method   | EM (HotpotQA) | F1 (HotpotQA) | TTFT       | E2E       | Ret/1K | Eff.↑    |
| -------- | ------------- | ------------- | ---------- | --------- | ------ | -------- |
| No-RAG   | 32.8          | 39.4          | 42 ms      | 2.3 s     | 0      | 17.1     |
| Sync-RAG | 69.2          | 75.1          | 287 ms     | 9.2 s     | 86     | 8.2      |
| PipeRAG  | 66.8          | 72.9          | 118 ms     | 5.6 s     | 66.8   | 13.0     |
| **Ours** | **68.7**      | **75.1**      | **108 ms** | **5.2 s** | **59** | **14.4** |
| Oracle   | 70.3          | 76.2          | 45 ms      | 3.1 s     | 48     | 24.6     |

- **TTFT** reduced by **62.4%** (287 ms → 108 ms)
- **E2E latency** reduced by **43.5%** (9.2 s → 5.2 s)
- **31% fewer retrievals** per 1K tokens (86 → 59)
- Answer quality within **0.5–0.9% EM** of synchronous RAG
- Retrieval Predictor AUROC: **0.81** (vs. 0.66 for entropy-only baseline; +22.7%)
- High-confidence predictions (p̂ > 0.8): **8.7 token lead time**, **78.3% hit rate**

**Code (RepoBench-P):** 
![[Predictive Prefetch.pdf#page=7&rect=309,315,543,447|Predictive Prefetch, p.7]]

**Summarization (QMSum):**
![[Predictive Prefetch.pdf#page=8&rect=52,622,295,729|Predictive Prefetch, p.8]]

**Cross-model:** Consistent 61.5–63.4% TTFT improvement across Llama, GPT-OSS (MoE), and Qwen families. MoE models achieve best efficiency due to sparse activation patterns.

---

# 6. Strengths

- Addresses synchronous blocking 
- Treats _when_ and _what_ as jointly learnable problems rather than heuristics 
- T5-small as query generator outperforms raw 8B LLM (QRS 0.79 vs. 0.74) --> task-specific fine-tuning outweighs scale, a practically useful finding.
- The OOD transfer result (pretrained AUROC 0.67–0.68 on code/summarization, exceeding the entropy-only 0.66 baseline) suggests the predictor learns properties of transformer computation rather than dataset-specific patterns.

---

# 7. Gaps

- **Closed-model inaccessibility:** Hidden states, attention weights, and value vectors are required. 
	=> Extending to parameter-efficient approximations for closed APIs (e.g., deriving uncertainty from output logits alone) is a meaningful research direction.
- **Training data requirement:** 50–100K generation traces are needed, posing a cold-start problem for new domains. 
- **Static threshold calibration:** Hyperparameters θ, Δ, and τ_rag require domain-specific tuning. 
	=> Automated calibration, potentially as a meta-learning problem over retrieval utility signals, would reduce deployment friction.

---

# 8. Highlights
> [!PDF|255, 208, 0] [[Predictive Prefetch.pdf#page=1&annotation=970R|Predictive Prefetch, p.1]]
> > a retrieval predictor, a context monitor, and a query generator,

> [!PDF|255, 208, 0] [[Predictive Prefetch.pdf#page=1&annotation=967R|Predictive Prefetch, p.1]]
> > enables predictive prefetching aligned with evolving information needs. 

> [!PDF|255, 208, 0] [[Predictive Prefetch.pdf#page=1&annotation=982R|Predictive Prefetch, p.1]]
> > Applications that demand high factual accuracy must tolerate multiple retrieval rounds and cumulative delays, whereas latency-sensitive deployments are forced to limit retrieval dept

> [!PDF|255, 208, 0] [[Predictive Prefetch.pdf#page=1&annotation=985R|Predictive Prefetch, p.1]]
> > refetching may introduce irrelevant context, undermining both generation efficiency and factual reliability. 

> [!PDF|255, 208, 0] [[Predictive Prefetch.pdf#page=2&annotation=988R|Predictive Prefetch, p.2]]
> > retrieval needs are preceded by identifiable semantic precursors in generation dynamics, such as characteristic patterns in entropy trajectories, attention allocation, and value representation dynamics, which emerge approximately 8–16 tokens before uncertainty becomes critical.

> [!PDF|255, 208, 0] [[Predictive Prefetch.pdf#page=2&annotation=991R|Predictive Prefetch, p.2]]
> > these same signals encode retrieval intent and can be leveraged to infer the retrieval query itself.
> 
> 