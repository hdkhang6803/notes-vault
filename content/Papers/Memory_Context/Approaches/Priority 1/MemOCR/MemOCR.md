---
Tags:
Date: "2026"
Authors: Yaorui Shi, Shugui Liu, Yu Yang, Wenyu Mao, Yuxin Chen, Qi Gu, Hui Su, Xunliang Cai, Xiang Wang, An Zhang
Venue: ICML
Paper: "MemOCR: Layout-Aware Visual Memory for Efficient Long-Horizon Reasoning"
Memory type:
  - Token-level
Agent env: 2-model pipeline
Record format: Text -> Image
Memory architecture:
Tackle Module: Context optimization
Need offline initialization: false
Fine-tuning?: true
Other tags:
---
# 1. Terminology

| Term                             | Definition                                                                                                                                                                                                          |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Memory budget (B)**            | The maximum number of tokens (text) or visual patch tokens (image) allowed in the working context at answer time                                                                                                    |
| **Adaptive information density** | The ability to allocate more context space to crucial evidence and less to auxiliary details, decoupling storage cost from word count                                                                               |
| **Uniform information density**  | The limitation of text-based memory where every token costs the same budget unit regardless of semantic importance                                                                                                  |
| **Rich-text memory (M^RT)**      | A structured Markdown memory that encodes visual priority via formatting cues (headings, bold, font size)                                                                                                           |
| **Visual memory (V_T)**          | The 2D image produced by rendering the rich-text memory; serves as the agent's sole working context at query time                                                                                                   |
| **Memory Drafting**              | Stage 1: the agent incrementally edits the rich-text memory, deciding what to keep and what to emphasize                                                                                                            |
| **Memory Reading**               | Stage 2: the agent reads the rendered memory image to answer a query                                                                                                                                                |
| **Budget-agnostic drafting**     | The drafter writes a complete memory without knowing the runtime budget; salience structure is baked into the layout                                                                                                |
| **Resolution manipulation**      | Downsampling the rendered image to control the number of visual patch tokens, providing a flexible budget-fidelity trade-off                                                                                        |
| **Visual salience**              | The perceptual prominence of a memory region determined by font size, weight, and layout position                                                                                                                   |
| **GRPO**                         | Group Relative Policy Optimization:  the RL algorithm used to jointly optimize drafting and reading                                                                                                                 |
| **$\mathcal{T}_{std}$**          | Standard QA training task: unmodified question + full 512-token memory budget                                                                                                                                       |
| **$\mathcal{T}_{augM}$**         | Augmented Memory task: original question + aggressively 4× downsampled (16× fewer pixels) memory image                                                                                                              |
| **$\mathcal{T}_{augQ}$**         | Augmented Question task: detail-oriented question + full uncompressed memory, ensuring auxiliary details are retained                                                                                               |
| **Aggregated advantage**         | Weighted combination of task-specific advantages used to update the drafting policy across all budget scenarios                                                                                                     |
| **SEM**                          | Sub-word Exact Match: accuracy metric which tokenizes both the predicted answer and the gold answer into sub-word tokens then checks whether the gold token sequence appears as a contiguous subsequence inside the |
| **Crucial region**               | High-priority layout area (e.g., H1 headers) that is more compression-robust and survives aggressive downsampling                                                                                                   |
| **Detailed region**              | Lower-priority layout area (e.g., plain body text) that is more susceptible to becoming unreadable under compression                                                                                                |
| Visual patch token               | 28 x 28 px/token<br>-> 16 tokens (12,544 px)<br>-> 1024 tokens (802,816 px)                                                                                                                                         |

---
# 2. Paper Summary (What)

MemOCR is a multimodal memory agent that re-frames agentic memory management as a **2D spatial allocation problem** rather than a 1D token-packing problem. Instead of serializing interaction history as plain text, MemOCR maintains a **structured rich-text (Markdown) memory** and renders it into a **memory image** that the agent reads at query time. 
By controlling visual layout (font size, heading level, bold emphasis) the agent can pack more content into far fewer visual tokens while keeping critical evidence readable even under aggressive resolution reduction. 

---
# 3. What it Solves (Why)

Every token costs the same budget unit regardless of semantic importance in prior memory paradigms (raw history injection and textual summarization)
- **Raw history memory** retrieves the top-k relevant passages but can be redundant and noisy, exhausting the budget on low-value content
- **Textual summary memory** compresses history but still couples storage cost to word count: keeping 100 tokens of crucial evidence forces retaining ~900 tokens of auxiliary details at the same density.
**=> Using visual layout as a non-uniform allocation mechanism:** crucial evidence rendered large and prominent survives downsampling; auxiliary details rendered small can be safely blurred away.

---

# 4. Methodology (How)
![[MemOCR.pdf#page=3&rect=312,580,513,685|MemOCR, p.3]]
MemOCR operates through a two-stage lifecycle separated by a deterministic rendering step.
## 4.1. Stage 1: Memory Drafting (Text Domain)

At each time step $t$, upon receiving a new text chunk $C_t$, the LLM agent updates a persistent Markdown memory $M^{RT}_t$:

$$
M^{RT}_t \approx \pi_{\theta}(·| M^{RT}_{t-1}, C_t)
$$
The agent acts as a **memory drafter**: it decides _what_ to keep and *what* to emphasize.
- **Crucial evidence** → H1/H2 headings, bold, larger effective font size.
- **Auxiliary details** → plain body text, smaller effective size.

Crucially, the drafter **never sees the runtime budget $B$**; it writes a single rich-text memory whose internal salience structure handles all compression levels after rendering.

> Query: _"Who wrote 'Put Your Hand in the Hand'?"_ After reading relevant chunks, the drafter emits:
> 
> ```markdown
> # Ocean Band
> Recorded by Canadian singer Anne Murray.
> 
> # Gene MacLellan
> Canadian singer-songwriter. Composed "Put Your Hand in the Hand".
> ```
> 
> Entity names are H1 headers; biographical detail is body text.

---

## 4.2. Stage 2: Memory Reading (Vision Domain)

### 4.2.1. Rendering (Deterministic, No LLM)

After all _T_ chunks are processed, the final Markdown $M^{RT}_T$ is converted to a 2D image $V_T$ by a lightweight renderer (FastAPI + Playwright/Chromium):

$$
V_T = \mathcal{R}(M^{RT}_T)
$$
The budget $B$ is enforced **only here** by downsampling $V_T$ to at most $B$ visual patch tokens (28×28 px/token in Qwen2.5-VL). 

The area occupied by a text segment of length $L$ at font scale $s$  (approx visual token cost) scales as $O(L · s²)$, so larger-font crucial evidence survives downsampling while small-font details blur away gracefully.


> At _B_ = 1024 tokens (802,816 px): both headers and body text are legible. 
> At _B_ = 16 tokens (12,544 px): "# Gene MacLellan" as an H1 header remains recognizable; body text collapses to noise — but the key answer entity is preserved.

### 4.2.2. Reading
The agent receives the budgeted image $V_T$ (as working context) alongside the query $Q$ and generates the answer $A$:

$$
A \approx \pi_\theta(· | V_T, Q)
$$
No raw history or original long context is provided: all task-relevant information must be recovered from $V_T$.

> With the budget of 16 tokens, the VLM reads the downsampled image, recognizes "Gene MacLellan" in the large H1 header, and correctly answers the question whereas a textual agent hard-truncated to 16 tokens had already lost that entity from its context.

---
## 4.3. Training: Budget-Aware RL Objectives (GRPO)

To prevent a **shortcut policy** (rendering everything at uniform medium size, collapsing adaptive density back to uniform density), MemOCR is trained with three complementary tasks on the same drafted memory:

| Task                                              | Memory budget | Question source          | Purpose                                                                            |
| ------------------------------------------------- | ------------- | ------------------------ | ---------------------------------------------------------------------------------- |
| **$\mathcal{T}_{std}$**: Standard QA              | 512 tokens    | Original question        | Global QA correctness at moderate budget                                           |
| **$\mathcal{T}_{augM}$**: QA w/ Augmented Memory  | 32 tokens     | Original question        | Force the model to focus on crucial evidence to survive 4x downsampling            |
| **$\mathcal{T}_{augQ}$** QA w/ augmented question | 512 tokens    | Detail-oriented question | Force model to identify low-priority fine-grained details when explixitly required |

**Reader** is updated with **separate** task-specific advantages (Avantage: A reward/penalty given to a model's output by comparing to other outputs in the sampling group). 
**Drafter** is updated with an **aggregated** advantage:

$$
A = \frac{\Sigma_k (w_k · A^{(k)})}{\Sigma_k w_k},    k ∈ {T_{std}, T_{augM}, T_{augQ}}
$$

with weights _w_ = {1.0, 0.7, 0.3} respectively.

**Running example:**

> Under T_augM, the 16-token image shows only the H1 headers → the drafter receives a reward signal only if "Gene MacLellan" is in a header. 
> 
> Under T_augQ, a detail question asks the recording artist → the drafter must also preserve "Anne Murray" somewhere legible at full resolution. The aggregated gradient teaches a layout where entity names are headers and supporting facts are body text.

---
# 5. Benchmarks

## 5.1. Other Baselines

**Raw History Memory (no compression)**
- **Qwen2.5-Instruct** — standard long-context LLM, supports up to 100K tokens.
- **R1-Distill Qwen** — Qwen model distilled from DeepSeek-R1; strong short-context reasoning but degrades catastrophically at 30K/100K.
- **Qwen2.5-1M-Instruct** — 1M-token context window model; strongest raw-history baseline.

**Textual Summary Memory**
- **[[Mem0]]** — production-ready agent with scalable long-term memory via incremental summarization.
- **[[Mem-α]]** — RL-trained hierarchical memory with multi-scale memory corpora.
- **[[MemAgent]]** — "memorize while reading" paradigm, distills task-relevant information chunk-by-chunk via RL (strongest text-based baseline).

**RAG-based Memory (unlimited external storage)**

- **SimpleMem**, **LightMem**, **HyMem** — retrieve relevant passages at inference time from an unbounded external store.

**Scaling ablations (no RL)**

- Qwen2.5-VL-7B / 32B / 72B-Instruct — larger backbones without budget-aware RL, used to test whether scaling substitutes for learned layout policy.

## 5.2. Benchmarks

| Benchmark                   | Type                            | Context lengths evaluated |
| --------------------------- | ------------------------------- | ------------------------- |
| [[HotpotQA]]                | Multi-hop QA                    | 10K / 30K / 100K tokens   |
| [[2WikiMultiHopQA]] (2Wiki) | Multi-hop QA                    | 10K / 30K / 100K tokens   |
| [[Natural Questions]] (NQ)  | Single-hop QA                   | 10K / 30K / 100K tokens   |
| [[TriviaQA]]                | Single-hop QA                   | 10K / 30K / 100K tokens   |
| [[LoCoMo]]                  | Long-term conversational memory | Zero-shot generalization  |

Memory budgets tested: B ∈ {16, 64, 256, 1024} tokens. Metric: sub-word exact match (SEM), averaged over 3 runs.

## 5.3. Notable Results

- **Overall accuracy (B = 1024, 10K original context):** MemOCR achieves 74.6% average accuracy, surpassing the strongest text baseline (MemAgent: 67.8%) by +6.8 points (p < 0.01).
- **Budget robustness - the key result:** MemOCR degrades only 16.6% relative at B = 16 (10K context), versus MemAgent's 53.3% drop. At 8 tokens, MemOCR matches baselines at 64 tokens → **8× token-efficiency gain**.
- **vs. RAG with unlimited storage:** MemOCR at B = 64 (67.3%) outperforms HyMem with unlimited storage (56.4%) by +10.9 points at 10K context.
- **Budget fairness:** MemOCR at B = 16 (62.2%) outperforms MemAgent at B = 64 (50.7%) — i.e., MemOCR with 4× fewer tokens still wins.

---

# 6. Strengths

- **Adaptive density through a free variable.** Rendering resolution is independent of memory content; budget can be adjusted post-hoc without re-running the drafter, enabling zero-cost budget flexibility.
- **Cross-model generalization.** The layout policy transfers to unseen VLM readers (GPT-4o) and out-of-distribution conversational data (LOCOMO), suggesting the typographic hierarchy is a universal signal rather than a model-specific artifact.
- **Multi-hop training transfers to single-hop.** Training on HotpotQA improves NQ as well, whereas single-hop training on NQ degrades HotpotQA -> complex drafting policies generalize asymmetrically.

---

# 7. Gaps

- **Content-length overflow pathology.** Repetitive generation can produce rich-text memories that force the renderer to use sub-threshold font sizes across the entire canvas (Failure Mode B). A promising direction is **memory length regularization** during RL
	=> a token-count penalty in the reward, or a hard max-length constraint enforced differentiably.
- **Static Markdown as the only format.** The current rendering pipeline is restricted to Markdown's formatting primitives. Richer formats (HTML/CSS, SVG) could enable semantically richer spatial layouts 
	=> side-by-side comparison tables, color-coded evidence categories, or sparkline summaries for temporal data 
- **Layout policy is domain-specific:** The trained salience strategy (entity names as headers, supporting facts as body text) may not generalize to workloads where crucial evidence is relational, procedural (tool-use logs), or sequential (planning traces). 
- **No lifelong or online update mechanism.** The current pipeline processes a finite chunk stream and produces a static final memory. For truly persistent agents, memory must handle deletions, corrections, and growing entity graphs over unbounded time.

---

# 8. Highlights
> [!PDF|] [[MemOCR.pdf#page=1&selection=55,16,57,3|MemOCR, p.1]]
> > allocating memory space with adaptive information density through visual layout

> [!PDF|255, 208, 0] [[MemOCR.pdf#page=1&annotation=753R|MemOCR, p.1]]
> > At the core of longhorizon reasoning is memory management under a finite working context: agents must continually decide what past information to store and what to retrieve into the context window

