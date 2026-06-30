---
Tags:
Date: "2026"
Authors: Ryan Wei Heng Quek, Sanghyuk Lee, Alfred Wei Lun Leong, Arun Verma, Alok Prakash, Nancy F. Chen, Bryan Kian Hsiang Low, Daniela Rus, Armando Solar-Lezama
Venue: CATS@ICML26 (under review)
Paper: "MEMO: Memory as a Model"
Memory type:
  - Parametric
Agent env: 2-model pipeline
Record format: Parameters
Memory architecture:
  - LLM as Memory
Tackle Module:
  - Memory management
Need offline initialization: true
Fine-tuning?: true
Other tags:
---
# 1. Terminology

|                                    |                                                                                                                                                          |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Reflection**                     | A synthesized QA pair derived from a corpus that encodes knowledge in compositional form, serving as the shared interface between training and inference |
| **Memory Model (Mφ)**              | A small, dedicated LM (e.g., 14B) fine-tuned on reflection QA pairs to parametrically internalize corpus knowledge                                       |
| **Executive Model (Mθ)**           | A frozen LLM (e.g., 32B or proprietary) responsible for reasoning and decomposing user queries; treated as a black box                                   |
| **Generator Model (Mgen)**         | An LLM used only during data synthesis to distill the target corpus into reflections; may be smaller than or the same as the Executive Model             |
| **Target Corpus (D)**              | The set of documents containing domain-specific or up-to-date knowledge that the frozen Executive Model cannot reliably recall                           |
| **Cross-document synthesis**       | The process of constructing QA pairs whose answers require integrating evidence from multiple documents                                                  |
| **Reversal curse**                 | The failure of LLMs trained on "A is B" to generalize to "B is A"; mitigated by entity-surfacing pairs                                                   |
| **Task vector (τ)**                | The parameter-space delta between a fine-tuned Memory Model and the base model; used for model merging                                                   |
| **Retrieval noise**                | Distractor (negative) documents in the corpus that are irrelevant to a query but degrade retrieval-based methods                                         |
| **Representation coupling**        | The tight binding of latent memory representations to the specific model that produced them, preventing cross-LLM reuse                                  |
| **TIES / DARE**                    | Model merging methods that resolve inter-task-vector interference via sparsification and sign-conflict resolution                                        |
| **Structured multi-turn protocol** | The three-stage inference pipeline (Grounding → Entity Identification → Answer Synthesis) through which the Executive Model queries the Memory Model     |

---

# 2. Paper Summary (What)

MEMO (Memory as a Model) is a modular framework for integrating new or domain-specific knowledge into a frozen LLM without modifying its parameters:
- A dedicated **Memory Model** is fine-tuned on a synthesized QA dataset derived from the target corpus, while a frozen **Executive Model** queries it at inference time via a structured multi-turn protocol. 

---

# 3. What it Solves (Why)

Existing approaches for knowledge integration each suffer from at least one critical failure mode:

- **Non-parametric methods** (RAG, ICL) are constrained by context window limits, scale poorly with corpus size, and are highly sensitive to retrieval noise. They also struggle to synthesize information distributed across multiple documents.
- **Parametric methods** (CPT, SFT) are expensive to re-run, prone to catastrophic forgetting, tend to memorize training distributions rather than generalize, and are infeasible for closed-source models.
- **Latent memory methods** (AutoCompressor, Gist tokens, ICAE) compress knowledge into soft tokens but suffer from representation coupling (the memory is inseparable from the model that produced it).

MEMO addresses all five desiderata simultaneously:

|Property|RAG / ICL|CPT / SFT|Latent Memory|MEMO|
|---|---|---|---|---|
|Frozen base LLM|✓|✗|✓|✓|
|No retrieval index|✗|✓|✓|✓|
|Black-box compatible|✓|✗|✗|✓|
|No catastrophic forgetting|✓|✗|✓|✓|
|Constant-size memory|✗|✓|✗|✓|
|Cross-LLM transferable|✓|✗|✗|✓|

---

# 4. Methodology (How)
![[MeMo.pdf#page=2&rect=105,612,506,726|MeMo, p.2]]
MEMO operates in two phases: 
- a **training phase** that builds the Memory Model from a target corpus
- an **inference phase** in which the Executive Model queries the Memory Model.

> **Running example:** Suppose the target corpus D is the full text of _NarrativeQA_  long books where a user later asks: _"Who is Linda to Earl?"_ 
## 4.1. Data Synthesis Pipeline (Training Phase)

Given corpus $D$, a Generator Model ($M_{gen}$) produces a reflection QA dataset $Q$ through five sequential steps:

**Step 1: Fact Extraction:** 
- Each document is chunked (e.g., sliding windows of 6,400 words for long novels).
- For each chunk, $M_{gen}$ performs two parallel extractions: 
	- **direct** (explicitly stated facts)
	- **indirect** (inferred or synthesized information beyond the surface text)
	- Both sets are merged into $Q_{raw}$.

> From a chapter describing Earl's illness:
> 	Direct extraction yields Q: "What is Earl's medical condition?" → A: "terminal cancer." 
> 	Indirect extraction yields Q: "Who is responsible for Earl's care at home?" → A: "His wife Linda."

**Step 2: Consolidation of redundant or overlapping information:** 
- $M_{gen}$ identifies QA pairs sharing a common context (entity, time period, relationship) and merges them into multi-fact pairs ($Q_{mrg}$) (requiring to integrate multiple facts rather than answer single-hop queries)
- The consolidated set is $Q_{con} = Q_{dir} ∪ Q_{indir} ∪ Q_{mrg}$.

> The two pairs above are merged into Q: "Who is the wife and caregiver of Earl during his terminal illness?" → A: "Linda."

**Step 3: Verification and Rewriting:**
- Each pair in $Q_{con}$ is tested for **self-containment** (can it be answered without the source chunk) 
- Pairs with unresolved pronouns (e.g., "What did _they_ do?") or implicit references are rewritten using the source chunk as context
- Still irresolvable pairs are discarded. This yields $Q_{ver}$.

>  "What did she do for him?" is rewritten to "What did Linda do for Earl during his illness?" → A: "She served as his primary caregiver." If still ambiguous, it is discarded.

**Step 4: Entity Surfacing:** 
- For each named entity in $Q_{ver}$, $M_{gen}$ generates QA pairs ($Q_{ent}$) whose questions encode the entity's attributes and relationships in the question and answer reveal the entity's identity. 
- This directly targets the **reversal curse**: training the Memory Model to infer an entity from an indirect description rather than only from its name.

> Q: "Earl's wife, who cared for him through his final illness, is named...?" → A: "Linda." This reversal of the standard direction enables inference-time entity identification.

**Step 5: Cross-Document Synthesis:** 
- Over pre-defined document groups $G_i$ (group = all chunks fom the same document or human-provided labels), $M_{gen}$ constructs QA pairs requiring evidence from multiple chunks. 
- Two connection types are targeted:
	- **converging clues**: multiple documents supply complementary facts about the same entity
	- **parallel properties**: different entities across documents share a common role or attribute, enabling analogical reasoning. 
- This is the most critical step: its removal collapses accuracy by ~75% on NarrativeQA.

> 	converging clues: Chapter 3 establishes that Earl is married; Chapter 7 names Linda as his caregiver; Step 5 synthesizes Q: "Who is Linda to Earl?" → A: "Linda is Earl's wife and caregiver during his final illness."

The final dataset is **$Q_{final} = Q_{ver} ∪ Q_{ent} ∪ Q_{cross}$**.

---

## 4.2. Training the Memory Model

- The Memory Model $M_φ$ initialized from a small pretrained LM, e.g., Qwen2.5-14B-Instruct
- It is fine-tuned via SFT on $Q_{final}$, minimizing next-token prediction loss over answer tokens only:

$$\mathcal{L}(\phi) = -\sum_{(q_i, a_i) \in Q_\text{final}} \sum_{t=1}^{|a_i|} \log M_φ\left(a_i^{(t)} \mid q_i, a_i^{(1:t-1)}\right)$$

- Conditioning only on the question (never the source chunk) forces parametric internalization of knowledge. 
- The Memory Model is strictly smaller than the Executive Model (e.g., 14B vs. 32B) and serves as a compact, queryable knowledge store.

## 4.3. Continual Knowledge Integration via Model Merging

- When a new corpus arrives, MEMO avoids retraining from scratch by using **model merging**. 
- A new Memory Model is independently trained on the new corpus, and its task vector $τ_i = φ_i − φ_0$ is merged with existing task vectors in parameter space using different methods:
	![[MeMo.pdf#page=21&rect=131,67,513,243&color=yellow|MeMo, p.21]]
- The best-performing method is TIES (ρ=0.3), which:
	1. trims each task vector ($τ_i$) to its top-ρ largest-magnitude entries
	2. resolves sign conflicts via magnitude-weighted majority vote
	3. merges only entries agreeing with the elected sign. 
Example:
		
| Param | φ₀ (base) | φ₁ (medical) | φ₂ (legal) | τ₁       | τ₂       |
| ----- | --------- | ------------ | ---------- | -------- | -------- |
| p1    | 1.0       | 1.8          | 0.4        | **+0.8** | **−0.6** |
| p2    | 2.0       | 2.5          | 2.9        | **+0.5** | **+0.9** |
| p3    | 0.5       | 0.6          | 0.4        | **+0.1** | **−0.1** |
| p4    | 3.0       | 3.3          | 3.6        | **+0.3** | **+0.6** |
At p = 0.3, keep top 30% of entries for each vector:

| Param | τ₁    | τ₂    | Keep in τ₁ (top 30%)? | Keep in τ₂ (top 30%)? |
|-------|-------|-------|-----------------------|-----------------------|
| p1    | 0.8 ✓ | 0.6 ✓ | yes                   | yes                   |
| p2    | 0.5   | 0.9 ✓ | no                    | yes                   |
| p3    | 0.1   | 0.1   | no                    | no                    |
| p4    | 0.3   | 0.6   | no                    | yes                   |
Resolve sign conflict:

| Param | Active entries   | Magnitude-weighted vote      | Elected sign |
| ----- | ---------------- | ---------------------------- | ------------ |
| p1    | τ₁=+0.8, τ₂=−0.6 | +0.8 vs −0.6 → positive wins | **+**        |
| p2    | τ₂=+0.9 only     | +0.9                         | **+**        |
| p3    | none             | —                            | 0            |
| p4    | τ₂=+0.6 only     | +0.6                         | **+**        |
Merge:

|Param|Elected sign|τ₁ agrees?|τ₂ agrees?|Merged delta|φ_merged|
|---|---|---|---|---|---|
|p1|+|+0.8 ✓|−0.6 ✗|+0.8|**1.8**|
|p2|+|0 (trimmed)|+0.9 ✓|+0.9|**2.9**|
|p3|0|—|—|0|**0.5**|
|p4|+|0 (trimmed)|+0.6 ✓|+0.6|**3.6**|


---

## 4.4. Structured Multi-Turn Inference Protocol

At inference time, the frozen Executive Model $M_θ$ queries $M_φ$ through a three-stage protocol, treating it as a black-box.
Complex queries are decomposed into targeted sub-queries aligned with the reflection interface.

> _Example query:_ "Who is Linda to Earl?"

**Stage 1: Grounding:** 
- $M_θ$ decomposes the query into atomic sub-questions, each targeting a single identifying constraint. 
- $M_φ$ answers each independently.

> Sub-questions: 
> - "Who is Earl?" → "Earl is an elderly man in a terminal illness."
> - "Does Earl have a spouse?" → "Yes, his wife is Linda."

**Stage 2: Entity Identification:** 
- Using grounding responses as context, $M_θ$ iteratively issues follow-up sub-queries to $M_φ$ to narrow the candidate entity set. 
- This stage leverages the entity-surfacing pairs (Step 4). 
- An uncertain-answer streak tracker feeds each candidate's accumulated failure count into the entity-pinning prompt => Executive Model progressively prioritize candidates that the Memory Model consistently cannot corroborate
- The stage terminates when a single entity e* is confirmed or the budget (7 interactions) is exhausted.

> $M_θ$ asks: "Is Linda Earl's wife?" → $M_φ$ answers: "Yes, Linda is Earl's wife." Entity confirmed: e* = Linda.

**Stage 3: Answer Synthesis:** 
- Conditioned on e*, $M_θ$ issues targeted follow-up questions to gather supporting facts, then synthesizes the final answer:

$$\hat{a} = M_\theta\left(q, \{m_j\}_{j=1}^J, e^*, m_\text{seek}\right)$$

- All Memory Model responses are compact natural-language snippets whose length is independent of corpus size, ensuring constant-time inference regardless of how large D is.

> Final answer: "Linda is Earl's wife and caregiver during his final illness."


---

# 5. Benchmarks

## 5.1. Other Baselines

| Baseline              | Type                   | Key Limitation                                       |
| --------------------- | ---------------------- | ---------------------------------------------------- |
| [[BM25]]              | Lexical retrieval      | Cannot handle semantic or multi-hop queries          |
| [[NV-Embed-V2]]       | Dense retrieval        | Sensitive to retrieval noise; no cross-doc reasoning |
| [[HippoRAG2]]         | Graph-based RAG (SOTA) | Noise-sensitive; constrained by context window       |
| [[Cartridges]]        | Trained KV-cache       | Requires white-box access to Executive Model         |
| [[Perfect Retrieval]] | Oracle upper bound     | Evidence documents directly in context               |

## 5.2. Benchmarks

| Benchmark               | Focus                                          | Size Used                        |
| ----------------------- | ---------------------------------------------- | -------------------------------- |
| **[[BrowseComp-Plus]]** | Multi-hop, multi-document deep research        | 300 questions, 3,541 documents   |
| **[[NarrativeQA]]**     | Discourse understanding over novels / scripts  | 293 questions, 105 documents     |
| **[[MuSiQue]]**         | 2–4 hop compositional reasoning over Wikipedia | 1,000 questions, 5,296 documents |

## 5.3. Notable Results

- **NarrativeQA:** MEMO (14B Memory Model, Gemini-3-Flash Executive) achieves 53.58% vs. the next-best baseline HippoRAG2 at 23.21% — a +30pp gap.
- **MuSiQue:** MEMO achieves 48.30% (Qwen2.5-32B-I) and 60.20% (Gemini-3-Flash), consistently outperforming all baselines.
- **BrowseComp-Plus:** MEMO leads with Gemini-3-Flash (66.67%) and remains competitive with Qwen2.5-32B-I (54.22%), narrowly trailing HippoRAG2 (56.11%) — expected given BrowseComp-Plus favours raw document access.
- **Noise robustness:** Adding 1× negative documents causes MEMO to change by ≤1.77%, vs. drops of up to 6.22% for retrieval baselines.
- **Plug-and-play gains:** Swapping to a stronger Executive Model (Gemini-3-Flash) without retraining the Memory Model yields +12–27pp across benchmarks.
- **Model merging:** TIES (ρ=0.3) trails full retraining by 11pp but beats all retrieval baselines, at 33% lower compute cost for K=2.

---

# 6. Strengths

- **Truly black-box compatible:** MEMO accesses the Executive Model only via its input-output interface, enabling deployment with proprietary APIs (e.g., Gemini) without code changes or weight access.
- **Noise-robust by design:** Because the Memory Model internalizes knowledge parametrically and answers sub-queries directly, it is insulated from the corpus-level noise that degrades retrieval indices.
- **Constant inference cost:** Memory Model responses are short natural-language snippets independent of corpus size, unlike RAG whose latency scales with the retrieval index.
- **Cross-LLM transferability:** A single Memory Model trained once can be paired with any Executive Model at inference time; stronger reasoning models immediately yield better results.
- **Linear adaptation cost:** Model merging reduces cumulative adaption from Θ(K²) to Θ(K) while still outperforming all retrieval baselines.

---

# 7. Gaps

- **Memory Model capacity ceiling:** Performance is bounded by the representational capacity of the fixed-size Memory Model; sufficiently large or information-dense corpora may exceed what it can correctly compress.
- **Merging accuracy gap:** The best merging configuration (TIES ρ=0.3) still trails full retraining by 11–19pp, and the optimal merging recipe is task-dependent and currently selected by sweep rather than principled design.
- **Low adaptation to new corpus:** The current framework assumes a static target corpus; extensions to streaming or incrementally updated corpora (beyond the disjoint merging setting) remain unaddressed.
- **Stage budget sensitivity:** The optimal interaction budget (currently 7) per inference stage varies by task and Executive Model capability, yet is currently set without systematic tuning, leaving performance on the table.
- **RL for Memory Model training unexplored.** SFT memorizes distributions; post-training with RL (which generalizes better) has not been applied to Memory Model training, despite its demonstrated gains in related settings.

---

# 8. Highlights