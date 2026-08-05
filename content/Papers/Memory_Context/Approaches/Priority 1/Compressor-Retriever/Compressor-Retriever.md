---
Date: "2024"
Authors: Yuan Yang, Siheng Xiong, Ehsan Shareghi, Faramarz Fekri
Venue: arxiv
Paper: The Compressor-Retriever Architecture for Language Model OS
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
# 1. Terminology

| Term                                    | Definition                                                                                                                                                                                         |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| LM OS (Language Model Operating System) | Conceptual system in which an LLM acts as the CPU, the context window as RAM, and external tools/environment as I/O, requiring life-long statefulness across sessions.                             |
| Compressor                              | Module that maps a context segment **x** into compressed latent embeddings by appending special `<mem>` tokens and running the base LLM's forward pass under a segmented attention mask.           |
| Retriever                               | Module that encodes the current context into `<ret>` retrieval embeddings and performs top-down sparse self-attention search over the hierarchical database to gather relevant memories.           |
| Hierarchical Database                   | Multi-level store of compressed embeddings for past context, built by iteratively compressing segment **x** into a hierarchy [m̃₀, m̃₁, ..., m̃_L], with m̃₀ = **x**.                              |
| `<mem>` token (m)                       | Special learned token appended to context tokens; its output hidden state m̃ encodes the compressed information of the preceding segment.                                                          |
| `<ret>` token (r)                       | Special learned token appended to the current context; its output hidden state r̃ serves as the query/holder that gets populated with retrieved information.                                       |
| Compression factor (k)                  | Hyperparameter controlling how many tokens/embeddings at level _l_ are compressed into one embedding at level _l+1_; hierarchy depth L = ⌈log_k n⌉.                                                |
| Segmented Attention Mask (M)            | Modified causal mask restricting each `<mem>` token $m_j$ to attend only to its corresponding segment $[x_{k·(j−1)}, ..., x_{k·j}]$, so higher-level embeddings map to specific lower-level spans. |
| Top-down Sparse Retrieval               | Retrieval procedure that computes attention scores $a_l$ of memories m̃_l w.r.t. r̃_l via self-attention, then recurses from level L down to level 0, narrowing to relevant sub-branches.          |
| TopC(·)                                 | Function that selects the C highest-attention indices at level _l+1_ and gathers their corresponding child embeddings at level _l_, producing a pruned candidate set $m̃_{l,C}$ of size C·k.       |
| Compressor-Retriever Architecture       | Overall model-agnostic, end-to-end differentiable method combining compressor + hierarchical database + retriever, using only the base LLM's forward function, with no external modules.           |
| LoRA (Low-Rank Adaptation)              | Parameter-efficient fine-tuning method (rank r=8 used here) applied to all linear layers plus the `<mem>`/`<ret>` embeddings to elicit compression/retrieval behavior.                             |
| BPTT (Backpropagation Through Time)     | Gradient computation approach required here since compression/retrieval happen in latent space with no intermediate labels, forcing full unrolling of the recurrent forward calls.                 |
| ICL (In-Context Learning)               | Evaluation setting where few-shot examples are provided as context; used to test whether compressor-retriever can retrieve the "right" examples to match full-context performance.                 |

# 2. Method Summary (What)

Compressor-Retriever is a model-agnostic architecture for managing life-long context in an LLM-as-OS setting. Instead of adding standalone adapters or an external indexer (as in RAG), it repurposes the base LLM's own forward function for two roles: (1) a **compressor** that builds a coarse-to-fine hierarchical memory database of past context, and (2) a **retriever** that performs top-down, sparse self-attention search over that database to reconstruct only the relevant context at the needed granularity. Because both steps reuse the same forward pass (rather than introducing new networks), the whole pipeline remains end-to-end differentiable and can be jointly fine-tuned (via LoRA) with the base model.

# 3. What it Solves (Why)

The session-based interaction paradigm makes LLMs largely stateless across sessions because the context window is fixed in size, so long-lived context (past dialogs, tool logs, retrieved documents) cannot simply be kept in-window indefinitely.

- **Explicit text compression** (Prompt-SAW, LLMLingua): shrinks text but offers no structured, life-long retrieval mechanism across sessions.
- **Latent/recurrent compression** (Transformer-XL, AutoCompressor, ICAE, Recurrent Memory Transformer): compress and reuse embeddings, but only within a single session; no principled way to manage or retrieve context across sessions.
- **Retrieval-Augmented Generation (RAG)**: relies on a small external indexer model; its capacity is limited, it cannot be optimized jointly with the base model, and retrieval is not state- or task-dependent across different granularities.
- **=>** Compressor-Retriever uses _only_ the base LLM's forward function (no external modules) to build a hierarchical, coarse-to-fine memory database and perform top-down sparse self-attention retrieval — making compression and retrieval end-to-end differentiable and jointly trainable with the base model.

# 4. Methodology (How)
![[Compressor-Retriever.pdf#page=3&rect=104,540,506,719|Compressor-Retriever, p.3]]


> **Running example:** a past context segment $x = [x_1, ..., x_6]$ (three token chunks) needs to be stored, and later a new query needs to retrieve the relevant part of it.

**A. Memory/Database Structure (Compressor)**
- a collection of independent trees, one per past context chunk (chunk₀, chunk₁, chunk₂, ...) 
- **Per-chunk structure:** each chunk is stored as a tree of L+1 levels, `[m̃₀, m̃₁, ..., m̃_L]`:
    - **Level 0** = the finest granularity, the raw segment itself.
    - **Level L** (top) = the coarsest granularity, a single embedding summarizing the whole chunk.
    - Levels in between are progressively coarser summaries.
- **Depth:** `L = ⌈log_k(n)⌉`, where _n_ is the chunk's token length, k is branching factor --> tree depth scales logarithmically with chunk size
- **Node content:** every node at every level is a single embedding vector of the same dimensionality **d** 
- **Where it lives:** the whole tree, once built, is written to disk (outside the active context window) 
- **Size limits:** In their actual experiments, only a shallow, fixed instance of this structure was tested (50 nodes at level 1, collapsing to 1 at level 2).
- **No cross-chunk index:**  retrieval has to separately touch every chunk's root, since the database itself provides no shortcut across chunks


**B. Retrieval Flow (Retriever)**

- Retrieval (step B) can be triggered every generation turn, or the model can be instruction-tuned to emit a `<call_retrival>` token when it decides retrieval is needed.
- **Asynchronous retrieval**: rather than blocking generation, retrieval can run in a background process, with retrieved results gradually folded into the context as generation continues — trading a small quality/latency risk for responsiveness.
- Encode the current query context **x** together with appended `<ret>` tokens **r**: $[\_, r̃] ← f_{llm}([x, r])$, giving retrieval embeddings r̃ that will accumulate retrieved information.
- Starting at the top level $(l = L)$, self-attend the top-level memories against r̃ and record the attention weights $a_l$: 
	$a_l, [\_, r̃_{l-1}] ← f_{llm}([m̃_l, r̃_l])$.
- Use `TopC(·)` to keep only the C highest-attended indices at level l+1, and fetch their corresponding child embeddings at level l, forming a pruned set m̃_{l,C} (size C·k).
![[Compressor-Retriever.pdf#page=5&rect=238,427,377,470&color=yellow|LM-OS, p.5]]

- Repeat the self-attention + TopC pruning step down through the levels (l = L−1, ..., 0), so the search narrows level by level.


Level 3 (top):        m3_1
	            |
Level 2:        m2_1        m2_2
               |            |
Level 1:     m1_1 m1_2    m1_3  m1_4
	    |           |            |          |
Level 0:    x1,x2  x3,x4   x5,x6    x7,x8
> Say x5–x6 are about sports, x7–x8 also sports, and x1–x4 are about cooking. The query r̃ is "what's the score of last night's game?" — clearly sports-relevant.
> - **Start at the top.** Self-attend r̃ against m̃₃ = {m3_1}. There's only one node, so it's trivially kept. This gives attention weight a₃
> - **Expand to level 2.** `TopC` looks at a₃ and grabs m3_1's children: {m2_1, m2_2}. Self-attend r̃ against both. Since m2_2 covers the sports branch (x5–x8), it gets a much higher attention score than m2_1 (cooking branch).
> - **Prune with TopC (say C=1).** Keep only the top-1 attended node: m2_2. **Crucially, m2_1's entire subtree (m1_1, m1_2, and their children x1–x4) is never even touched**, no compute spent loading or attending to it.
> - **Expand to level 1.** Gather m2_2's children: {m1_3, m1_4} (both from the sports branch). Self-attend r̃ against these two. Suppose m1_3 (x5,x6: last night's specific game) scores higher than m1_4 (x7,x8: a different game).
> - **Prune again.** Keep only m1_3. Drop m1_4's branch (x7, x8) entirely.
> - **Expand to level 0 (bottom).** Gather m1_3's children: the actual fine-grained embeddings for x5, x6. Final self-attention pulls the specific content into r̃, which now carries the retrieved information.
    
- The search can early-stop once a coarse level is judged remotely relevant or once the desired granularity is reached.

**C. Updating / Inference Flow**

- Start with the raw segment: m̃₀ ← $x = [x_1, ..., x_6]$.
- Append a sequence of `<mem>` tokens **m** and run the forward pass under the segmented attention mask M, so each `<mem>` token only attends to its assigned chunk:
	 $[\_, m̃] ← f_{llm}([x, m], M)$.

>	Instead, they restrict each `<mem>` token to only attend to _its own slice_:
> 	- m₁ attends only to \[x₁, x₂\]
> 	- m₂ attends only to \[x₃, x₄\]
> 	- m₃ attends only to \[x₅, x₆]


- Repeat the compress step on the previous level's output to build the next level up: $[\_, m̃_{i+1}] ← f_{llm}([m̃_i, m_{i+1}], M_i)$ (i=0,1,...).

> _Example continued:_ $m̃_0=[x_1, ..., x_6]$ compresses into m̃₁ (one embedding summarizing each chunk), and m̃₁ further compresses into a single top-level embedding m̃₂, forming the hierarchy \[m̃₀, m̃₁, m̃₂\].
    
- This process runs over every past chunk of context, and the resulting hierarchies are stored on disk as the **hierarchical database**.

**D. Training**
- **Data unit:** each training sample = a target question q (with answer a) + 6 ICL examples attached (2 relevant to q's task, 4 irrelevant, drawn from other reasoning datasets).
- **Step 1- Build memory:** the compressor runs on the 6 ICL examples in the same forward pass, producing the hierarchical embeddings M = \[m̃₀, m̃₁, ...], built fresh per sample, not loaded from a stored database.
- **Step 2 - Retrieve:** the retriever takes q as the query, searches M via top-down sparse attention, and pulls back whichever ICL examples it judges relevant (ideally just the 2 matching ones).
- **Step 3 - Predict:** the model generates the answer using q plus whatever was retrieved, and computes the standard autoregressive loss on predicting a (and, more generally, every next token in the sequence).
- **Loss:** just next-token prediction, `L = -(1/n)Σ log p(x_{t+1}|x_t,...,x1, M)`  (no separate loss for compression quality or retrieval accuracy).
- **Backprop:** since compression + retrieval happen inside the same computation graph as the final prediction, gradients must flow all the way back through both — requiring full BPTT (no truncation), roughly 2L+2 forward passes before one backward pass.
- **Trainable parameters:** the `<mem>`/`<ret>` token embeddings, plus LoRA adapters (r=8) on all linear layers of the base model — not full fine-tuning.
# 5. Benchmarks

**5.1 Other baselines**

|Baseline|Description|
|---|---|
|0-shot|Base LLaMA3.1-8B-instruct with no ICL examples (lower-bound reference).|
|6-shot (full context)|All 6 ICL examples (2 relevant + 4 irrelevant) included directly, no compression/retrieval (ideal/upper-bound reference).|

**5.2 Benchmarks (datasets)**

| Dataset       | Task Type                                                                                     |
| ------------- | --------------------------------------------------------------------------------------------- |
| [[GSM8K]]     | Math word problems                                                                            |
| [[FOLIO]]     | Natural language inference (first-order logic) — held out as the _only_ test-time target task |
| [[proScript]] | Graph/procedural reasoning                                                                    |
| [[ReClor]]    | Commonsense/logical reading comprehension                                                     |

Each test sample mixes 2 relevant ICL examples (from the target task) with 4 irrelevant ones (from the other tasks), and the window is limited so the model must select a subset — testing whether it retrieves the relevant examples.

**5.3 Notable Results**

|Mode|Accuracy|
|---|---|
|0-shot|0.250|
|6-shot (full, ideal)|0.578|
|Compressor-Retriever|0.429|

- Compressor-Retriever reaches **~75% of full 6-shot ICL accuracy**, substantially above the 0-shot floor.
- Top-level attention analysis shows a **64% exact match rate** — i.e., in 64% of test cases the model's retrieved set contained all the correct (relevant) examples.

# 6. Strengths

- **Model-agnostic & minimally invasive**: adds only two special tokens (`<mem>`, `<ret>`) and a modified attention mask; applicable to any decoder-only transformer without extra modules.
- **End-to-end differentiable**: both compression and retrieval reuse the base model's own forward pass, so gradients flow through the entire pipeline — unlike RAG's frozen, separately-trained indexer.
- **Efficient sparse retrieval**: top-down `TopC` pruning avoids attending over the full database, enabling scalable coarse-to-fine search as the database grows.
- **Promising early evidence**: recovers ~75% of full-context ICL accuracy and a 64% exact retrieval-match rate from only a small-scale LoRA fine-tune on a single consumer GPU (RTX 4090).

# 7. Gaps

- **No true life-long/multi-session evaluation**: experiments only test single-session ICL example retrieval; the paper's core motivating scenario (statefulness across sessions, tool logs, web pages accumulating over time) is untested.
- **Narrow, fixed configuration**: uses one fixed compression scheme (50 embeddings at level 1, 1 at level 2) and one simple sequential segmentation strategy; alternative hierarchy depths/segmentations are unexplored.
- **Training data scarcity**: no native long-context training data is used; the method is validated on small ICL datasets rather than realistic long-document or tool-log corpora, sidestepping the data-curation challenge the paper itself flags.
- Can only apply to open models
- Manual case-by-case retrieval embedding size
- Attention-based retrieval is potentially not scalable
- **Proposed future works**: 
	- quantifying the latency/accuracy trade-off of asynchronous background retrieval during live generation
	- testing retrieval robustness and TopC pruning quality as the database scales to millions of tokens / thousands of chunks; 
	- comparing alternative segmentation schemes beyond fixed sequential chunking.

# 8. Highlights


> [!PDF|255, 208, 0] [[Compressor-Retriever.pdf#page=1&annotation=406R|LM-OS, p.1]]
> > One of the most important features of an OS is that it is forever stateful
> 
> 