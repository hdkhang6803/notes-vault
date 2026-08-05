---
Date: "2024"
Authors: Peng Wang, Zexi Li, Ningyu Zhang, Ziwen Xu, Yunzhi Yao, Yong Jiang, Pengjun Xie, Fei Huang, Huajun Chen
Venue: NeurIPS
Paper: "WISE: Rethinking the Knowledge Memory for Lifelong Model Editing of Large Language Models"
Memory type:
  - Parametric
Agent env: Single agent
Record format: Parameters
Memory architecture:
Tackle Module: Memory management
Need offline initialization: true
Fine-tuning?: false
Other tags:
---
# 1 Terminology

|Term|Definition|
|---|---|
|Long-term Memory|Knowledge stored directly in model parameters (pretrained knowledge); updatable via retraining, finetuning, or editing|
|Working Memory|Non-parametric knowledge in neural activations/representations, accessed via retrieval at inference time; does not change network parameters|
|Side Memory|WISE's proposed "mid-term memory" — a parametric copy of an FFN's value matrix, dedicated to storing edits|
|Main Memory|The original (unedited) FFN value matrix, holding pretrained knowledge|
|Reliability|The model correctly recalls both current and previous edits after sequential editing|
|Locality|Editing does not corrupt or influence unrelated pretrained knowledge|
|Generalization|The model understands edits well enough to answer paraphrases/related queries, not just memorized query-target pairs|
|Impossible Triangle|Core finding: no prior method achieves Reliability, Generalization, and Locality simultaneously under lifelong editing|
|Activation Routing (Δact)|Mechanism that decides, per query, whether to route through main memory or side memory|
|Knowledge Sharding|Splitting a stream of edits into _k_ subspaces via random gradient masks (mask ratio ρ) to control "knowledge density" and avoid conflicts|
|Knowledge Merging|Combining the _k_ edited subspaces into one side memory using Ties-Merge (trim → elect sign → disjoint merge)|
|WISE-Merge|Default variant: a single side memory, continuously updated via merging|
|WISE-Retrieve|Variant: multiple side memories retained, selected at inference via top-1 activation score routing|
|Knowledge Density|Ratio describing how many pieces of knowledge are stored per parameter on average|

| Method                    | How It Works                                                                                                                                                                                    |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **FT-L**                  | Freezes all layers except a single MLP layer, directly finetuned on the edit with an L∞ norm constraint capping parameter drift.                                                                |
| **FT-EWC**                | Direct finetuning + an Elastic Weight Consolidation penalty (Fisher-information-weighted) to reduce forgetting across sequential edits.                                                         |
| **ROME**                  | Causal tracing locates the fact-storing MLP layer, then overwrites its value matrix via closed-form least-squares — one fact at a time.                                                         |
| **MEMIT**                 | Extends ROME to distribute a _batch_ of edits across several MLP layers, solved jointly via least-squares for mass-editing.                                                                     |
| **MEMIT-MASS**            | MEMIT applied as one single-shot batch update over all edits at once, rather than incremental sequential repair.                                                                                |
| **MEND**                  | An offline-trained hypernetwork transforms raw finetuning gradients into smarter, low-rank targeted updates before applying them.                                                               |
| **DEFER** (SERAC reimpl.) | Frozen base LLM + a scope classifier (decides if a query is "in scope") + a small counterfactual model that answers in-scope queries.                                                           |
| **GRACE**                 | Frozen base LLM + a discrete key-value codebook per edit (key = last-token activation, value = edited output, with a shrinking "deferral radius"); matches queries by proximity to stored keys. |

---

# 2 Method Summary (What)

- WISE introduces a **dual parametric memory architecture**: 
	- a "main memory" preserving pretrained knowledge untouched
	- a "side memory" (a copy of one FFN layer's value matrix) where all edits are written.
- A trained **routing mechanism** decides, per query, whether to use the main or side memory. 
- To scale to thousands of sequential edits without conflict or catastrophic forgetting, WISE shards edits into random parameter subspaces and later **merges** them into a unified side memory using a Ties-Merge–based model-merging technique.

---

# 3 What it Solves (Why)

- **The impossible triangle problem**: Prior editing methods trade off between reliability, generalization, and locality. None achieve all three under sustained, sequential editing.
    - Long-term memory editing (ROME, MEMIT, FT-EWC): causes interference/conflicts with unrelated pretrained knowledge → **poor locality**, and degrades badly as edit count grows.
    - Working memory editing (GRACE, DEFER/SERAC): preserves locality and reliability well via retrieval, but the non-parametric/small auxiliary models **fail to generalize** to paraphrased or related queries.
- **Scalability to lifelong/continual editing**: Most prior methods were only validated on small numbers of edits (T=1–10); WISE is explicitly designed and tested for hundreds to thousands of sequential edits without retraining.

---

# 4 Methodology (How)

![[WISE.pdf#page=4&rect=111,564,497,732&color=yellow|WISE, p.4]]

> **Running example**: Suppose we sequentially edit an LLM (LLaMA-2-7B) with the fact _"Which continent is Berkner Island in? → South America"_, followed later by many more edits (e.g., _"Which language is Garowe Principles written in? → Persian"_), across a stream of T=1000 edits.

## 4.1 System / Memory Structure

- Each Transformer FFN layer computes $FFN(f) = a · W_v$, where `a` is the activation from the first MLP layer and $W_v$ is the value matrix (down-sampling matrix).
- WISE targets a single **mid-to-late layer** (e.g., layer 26–27 of 32 in LLaMA-2-7B) --> empirically shown to have the best trade-off (early layers encode low-level grammar; final layer directly controls decoding; mid-to-late layers integrate semantics with minimal side effects).
- At this layer, WISE **copies** the value matrix $W_v$ to create a **side memory** $W_{v'}$ (initialized identical to $W_v$). The original $W_v$ (main memory) is _never_ modified and represents pristine pretrained knowledge.

>So for our Berkner Island edit, instead of overwriting LLaMA's layer-27 $W_v$, WISE clones it into $W_{v'}$ and edits only the clone.

## 4.2 Retrieval Flow (Inference / Answering a Query)
> "Which continent is Berkner Island located on?" - a paraphrase of the original edit

1. Given an input query `x`:
	- compute the activation indicator: $Δact(x) = ‖A(x) · (Wv′ − Wv)‖₂$ 
	- This measures how much the side-memory edit would change the output relative to main memory, for this specific query.
2. **Routing decision**:
	- If $Δact(x) > ε$ (during editing, keep track of minimum activation indicator value as the threshold), the model uses the **side memory** $Wv′$ to compute the FFN output. 
	- Otherwise, it falls back to the **main memory** $Wv$ (pretrained, untouched knowledge)
	>	for an unrelated query like "Who won the golden boot?", locality is preserved because $Δact$ stays low and main memory is used.
3. **WISE-Retrieve variant**: 
	- If multiple side memories exist (rather than one merged memory), the model computes $Δact$ against _each_ side memory and routes to the one with the **maximal** activation score (top-1 retrieval among experts).
4. Output: $FFN_{out}(x) = A(x)·Wv′$ if routed to side memory, else $A(x)·Wv$.
## 4.3 Updating Flow (Writing an Edit)

1. **Sharding**: 
	- Before editing begins, generate _k_ random binary gradient masks (mask ratio ρ, e.g., ρ=0.2, k=2) over $Wv′$. 
	- Each mask defines a subspace of parameters that a given "shard" of edits is allowed to touch.
2. **Edit within a subspace**: 
	- For edit (xₑ, yₑ) = ("Berkner Island continent" → "South America"), gradients are computed from a combined loss: $L_{edit} = −log P(yₑ|xₑ) + L_a$ (an autoregressive loss plus a margin-based **routing activation loss**). 
	![[WISE.pdf#page=5&rect=118,660,483,695|WISE, p.5]]
	- Only the masked (active) parameters in the current subspace shard are updated; all others are frozen, preventing this edit from disturbing unrelated knowledge or previous shards.
3. **Routing threshold tracking**:
	- The smallest activation indicator $Δact(xₑ)$ seen among edits so far is tracked as ε
	- this becomes the "boundary" that later separates edit-relevant queries from irrelevant ones during inference.
4. **Shard filling → merging**: 
	- As more edits arrive (e.g., the Garowe Principles edit), they fill the next subspace shard. 
	- Once all _k_ shards are full, the _k_ subspace copies of $Wv′$ are merged into a single unified side memory using **Ties-Merge**: 
		1. trim redundant per-parameter updates
		2. elect the dominant sign per parameter
		3. compute a disjoint mean across shards with matching sign. This resolves conflicts in the small overlapping regions between subspaces (which also serve as useful "anchors" for fusion), while avoiding destructive interference.
5. This shard→edit→merge cycle repeats continuously as new edits stream in (or, in **WISE-Retrieve**, a _new_ side-memory copy is started once one fills up, rather than merging into the same one).


---

# 5 Benchmarks

## 5.1 Other Baselines

|Category|Method|Description|
|---|---|---|
|Long-term memory (finetuning)|FT-L|Finetunes a single MLP layer with L∞ norm constraint|
|Long-term memory (continual learning)|FT-EWC|Finetuning + Elastic Weight Consolidation to reduce forgetting|
|Long-term memory (locate-and-edit)|ROME|Causal-tracing-based single-fact edits via least-squares on MLP|
|Long-term memory (locate-and-edit)|MEMIT|Multi-layer, batched extension of ROME for mass editing|
|Long-term memory (locate-and-edit)|MEMIT-MASS|Batch (non-sequential) variant of MEMIT|
|Long-term memory (meta-learning)|MEND|Hypernetwork transforms finetuning gradients into low-rank targeted updates|
|Working memory (retrieval)|DEFER (SERAC reimpl.)|Scope classifier + small counterfactual model routes queries|
|Working memory (retrieval)|GRACE|Discrete key-value codebook with deferral radii, replaces activations at inference|

## 5.2 Benchmarks

| Setting            | Dataset      | Task                                                                                   | Size (T)      | Base Models            |
| ------------------ | ------------ | -------------------------------------------------------------------------------------- | ------------- | ---------------------- |
| QA                 | ZsRE         | Closed-book question answering edits                                                   | 1,000 / 3,000 | LLaMA-2-7B, Mistral-7B |
| Hallucination      | SelfCheckGPT | Correcting GPT-3-generated hallucinated passages with real Wikipedia text              | 600           | LLaMA-2-7B, Mistral-7B |
| OOD Generalization | Temporal     | Editing on emerging (post-2019) entities, evaluated on natural Wikipedia continuations | 10 / 75       | GPT-J-6B               |

Metrics used: 
- **Rel.** (Reliability / Edit Success Rate), 
- **Gen.** (Generalization Success Rate on paraphrases), 
- **Loc.** (Locality Success Rate on unrelated queries (unrelated queries' outputs unchange)),
- **Avg.** (mean of the three). 
![[WISE.pdf#page=7&rect=124,414,489,440|WISE, p.7]]
- Hallucination setting uses **Perplexity (PPL, lower=better)** for reliability and has no generalization metric.

## 5.3 Notable Results

- **QA (ZsRE), T=1000, LLaMA-2-7B**: WISE achieves **Avg. 0.83** vs. next-best MEMIT-MASS at 0.65 — an ~18-point improvement; Loc. stays at a perfect 1.00 throughout.
- **QA (ZsRE), T=1000, Mistral-7B**: WISE reaches **Avg. 0.79** vs. 0.68 for MEMIT-MASS.
- **Hallucination, T=600**: WISE maintains the lowest perplexity (3.12 on LLaMA, 5.21 on Mistral) while keeping Loc. ≥ 0.93, whereas ROME/MEMIT PPL explodes to 100+ and GRACE's PPL also degrades in long-text settings.
- **OOD (Temporal), GPT-J-6B**: WISE achieves the best overall average (0.78) and best OOD generalization (0.36–0.37), outperforming GRACE (0.75) and DEFER (0.29–0.36).
- **Scaling to 3K edits**: WISE-Retrieve (0.73 Avg.) and WISE-Merge (0.71 Avg.) both outperform GRACE (0.66) and MEMIT-MASS (0.53) at T=3000; an "oracle" routing upper bound reaches 0.82, showing headroom in retrieval accuracy.
- **Efficiency**: WISE-Merge adds only ~0.64% extra parameters and ~4% extra GPU VRAM, with constant (~3%) inference-time overhead — versus retrieval-memory methods that require ever-growing storage.
- **Ablations**: Removing the routing loss `L_a` drops Locality from 1.00 → 0.72 at T=1000; Ties-Merge substantially outperforms naive merging strategies (Linear/Slerp/Dare) — Avg. 0.87 vs. ~0.71–0.74 — confirming that resolving directional parameter conflicts during merge is critical.

---

# 6 Strengths

- Directly addresses and empirically resolves the **impossible triangle** (Reliability, Generalization, Locality) that all compared baselines suffer from.
- **Architecture-agnostic**: works as a modification to standard FFN layers, requiring no redesign of the base Transformer 
- **Scales gracefully** to thousands of sequential edits (T up to 3,000) with near-constant, small computational/memory overhead (~0.64% extra parameters, ~4% VRAM).
- **Non-destructive and reversible**: edits live in a separate side memory, so they can be discarded without harming the base model 
- **Strong empirical support:** ablations (layer selection, ρ/k hyperparameters, merge strategy, routing loss) are thorough and each design choice is independently justified.
- **Provides a theoretical guarantee (Theorem 2.1) on subspace overlap,** grounding the sharding/merging design rather than relying purely on empirics.

---

# 7 Gaps

- **Retrieval accuracy degrades with scale**: WISE-Retrieve's top-1 routing accuracy drops to ~60% at T=3000 (vs. an oracle upper bound of ~0.82 Avg.), indicating the side-memory selection mechanism is not yet reliable at very large edit counts.
- **Single-layer bottleneck**: WISE edits only one FFN layer per side memory; capacity is finite (paper notes ~500 edits per 20% of FFN parameters), requiring new side memories ("mask memory exhaustion") as scale grows
- **Imperfect fitting on rare/long-tail entities**: Case studies show factual failures on rare entities (e.g., "Persian" mis-predicted as "Dutchian") and partial-token errors, suggesting edit fitting is not fully robust.
- **Sensitivity to instruction phrasing**: WISE can correctly answer a paraphrase while failing the original edit prompt (or vice versa), suggesting the routing/generalization behavior is not fully consistent across semantically equivalent instructions.
- **Can only apply on open models**
- **Did not compare to some semantic retrieval based approaches**: it's built largely on the empirical weakness of GRACE/DEFER on the generalization axis. If that weakness is partly an implementation artifact (crude last-token keys, undersized auxiliary model) rather than a fundamental property of _all_ working-memory/retrieval approaches, the triangle's generality is somewhat overstated

**Proposed future directions:**

- Improve side-memory retrieval specificity (the paper's proposed `L_memo` replay constraint, which raised retrieval accuracy from ~60% to ~88% at T=3K) — worth exploring as a default rather than an ablation.
- Investigate **multi-layer or hierarchical side memories** to increase total edit capacity without proportionally increasing routing complexity.
- Extend evaluation to **mixed-domain edit streams** (not just single-dataset edits) to better stress-test routing specificity in realistic deployment (edits arriving from diverse sources/topics).
- Explore **automatic/adaptive layer selection** instead of a fixed heuristic (e.g., layer 26/27), potentially per-model or per-domain.

---

# 8 Highlights

