---
Tags:
Date: "2025"
Authors: Wujiang Xu, Zujie Liang, Kai Mei, Hang Gao1, Juntao Tan1, Yongfeng Zhang
Venue:
Paper: "A-Mem: Agentic Memory for LLM Agents"
Memory type:
  - Token-level
Agent env: Single agent
Record format: Text
Memory architecture:
  - Graph
Tackle Module: Memory management
Need offline initialization: false
Fine-tuning?: false
Other tags:
  - Self-evolving memory
---
i
# 1. Terminology

| Term                   | Definition                                                                                                                                         |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zettelkasten           | A knowledge management method that organizes information as atomic, interlinked notes forming an emergent knowledge graph                          |
| Memory Note ($m_i$)    | The structured unit of storage in A-MEM, containing content, timestamp, LLM-generated keywords/tags/context, a dense embedding, and a set of links |
| Note Construction      | The step that transforms a raw interaction into a full memory note by prompting an LLM to generate keywords, tags, and a contextual description    |
| Link Generation        | The step that connects a newly created note to semantically related historical notes using embedding similarity + LLM-driven relationship analysis |
| Memory Evolution       | The step that updates the keywords, tags, and context of existing notes when a new, related memory reveals higher-order patterns                   |
| Box                    | A cluster of mutually linked notes sharing similar contextual descriptions; one note can belong to multiple boxes simultaneously                   |
| Top-k Retrieval        | Selecting the _k_ most semantically similar notes to a query via cosine similarity over dense embeddings                                           |
| Memory Evolution Agent | The LLM acting autonomously to decide whether and how to update neighbor notes upon integration of a new memory                                    |

---

# 2. Paper Summary (What)

A-MEM introduces an agentic memory system for LLM agents that organizes memories dynamically  without relying on predefined schemas or fixed workflows. Inspired by the Zettelkasten method, each new interaction is stored as a structured note containing LLM-generated keywords, tags, a contextual description, and a dense embedding. Upon ingestion, the system autonomously identifies semantically related historical notes and creates links between them. Crucially, these new connections can also trigger backward updates to the context and attributes of existing notes, allowing the memory network to continuously refine itself as the agent accumulates experience. 

---
# 3. What it Solves (Why)

Existing retrieval-based memory system relies on fixed memory records
--> poor generalization across task types and limited effectiveness in long-term interactions.

---
# 4. Methodology (How)
![[A-Mem.pdf#page=3&rect=106,531,504,724&color=yellow|A-Mem, p.3]]

> **Running example.** Consider two consecutive conversations: 
> 
> _(Conv-1)_ "Can you help me implement a custom cache system for my web application? I need it to handle both memory and disk storage." 
> 
> _(Conv-2, later)_ "The cache system works great, but we're seeing high memory usage in production. Can we modify it to implement an LRU eviction policy?" 

## 4.1 Note Construction

Each interaction is converted into a memory note ($m_i = {c_i, t_i, K_i, G_i, X_i, e_i, L_i}$ )  
First, the Keywords, Tags and Context are retrieved by prompting an LLM with prompt template $P_{s1}$:
$$K_i, G_i, X_i \leftarrow \text{LLM}(c_i | t_i | P_{s1})$$
A dense embedding is then computed over the concatenation of all textual fields:
$$e_i = f_{\text{enc}}[\text{concat}(c_i, K_i, G_i, X_i)]$$

>**Example.** Conv-1 produces note $m_1$:
>
>- **Content** $c_1$: raw Conv-1 text
>- **Keywords** $K_1$: `[cache, web application, memory storage, disk storage, implementation]`
>- **Tags** $G_1$: `[software engineering, caching, system design]`
>- **Context** $X_1$: _"User requests implementation of a dual-storage cache for a web application."_
>- **Embedding** $e_1$: dense vector of the above concatenation
---
## 4.2 Link Generation

When note $m_n$ (from Conv-2) is added, its embedding is compared against all existing notes using cosine similarity:

$$s_{n,j} = \frac{e_n \cdot e_j}{|e_n||e_j|}$$

The top-$k$ nearest neighbors are identified:

$$M^n_{\text{near}} = {m_j \mid \text{rank}(s_{n,j}) \leq k,\ m_j \in M}$$

An LLM then decides which of these neighbors to link based on shared attributes and contextual alignment:

$$L_i \leftarrow \text{LLM}(m_n | M^n_{\text{near}} | P_{s2})$$

>**Example.** The embedding of Conv-2's note $m_2$ (LRU eviction, memory pressure) is cosine-similar to $m_1$ (cache implementation). 
>	=>The LLM confirms the connection: both concern the same web cache system across sequential stages of development.  
>	$m_2.L$ is updated to include a link to $m_1$, forming a box around "cache system development."

---
## 4.3 Memory Evolution

After linking, A-MEM re-examines each neighbor $m_j \in M^n_{\text{near}}$ and decides whether to update its context, keywords, or tags to reflect knowledge gained from $m_n$:

$$m^*_j \leftarrow \text{LLM}(m_n | M^n_{\text{near}} \setminus m_j | m_j | P_{s3})$$

The evolved note $m^*_j$ replaces the original in $M$.

>**Example.** $m_1$ originally describes a cache implementation request in isolation. After ingesting Conv-2, the LLM evolves $m_1$:
>
>- **Updated Context** $X^*_1$: _"User is building a production web cache; initial dual-storage implementation later required LRU eviction to manage memory pressure."_
>- **Updated Keywords** $K^*_1$: `[cache, web application, dual storage, LRU, production, memory pressure]`
>
The note now captures the full arc of the cache development thread rather than only the first turn.
---
## 4.4 Memory Retrieval

At inference time, a query $q$ is embedded and top-$k$ notes are retrieved by cosine similarity:

$$e_q = f_{\text{enc}}(q), \quad M_{\text{retrieved}} = {m_i \mid \text{rank}(s_{q,i}) \leq k,\ m_i \in M}$$

Because linked notes share a box, retrieving any one member automatically gets the entire linked cluster.

> **Example.** For the question _"What LRU policy did we design for the cache?"_, the query embedding is closest to $m_2$. Since $m_2$ is linked to $m_1$ within the cache box, both notes are returned. The agent receives the full context (original design intent, disk/memory dual storage, and the LRU evolution) enabling a precise, multi-hop answer without scanning all memories.

---

# 5. Benchmarks

## 5.1. Other Baselines

| Baseline           | Core Mechanism                                                                                                   |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| No memory          | No memory; stuffs the full preceding conversation into the prompt for each query                                 |
| **[[ReadAgent]]**  | Three-stage pipeline: paginate long context → gist each page → interactive look-up                               |
| **[[MemoryBank]]** | Dynamic memory with Ebbinghaus forgetting-curve-based strength updates; builds a user portrait over time         |
| **[[MemGPT]]**     | Dual-tier virtual context (RAM-like main context + disk-like external context) inspired by OS memory hierarchies |

## 5.2. Benchmarks

| Dataset         | Type                        | Scale                                                            | Task Categories                                               |
| --------------- | --------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------- |
| **[[LoCoMo]]**  | Long-term conversational QA | 7,512 QA pairs; ~9K tokens/dialogue; up to 35 sessions           | Multi-Hop, Temporal, Open Domain, Single-Hop, Adversarial     |
| **[[DialSim]]** | Multi-party dialogue QA     | ~350K tokens; 1,300 sessions (5 years); 1,000+ questions/session | General QA over TV-show dialogues (Friends, TBBT, The Office) |

**Evaluation metrics:** F1, BLEU-1, ROUGE-L, ROUGE-2, METEOR, SBERT Similarity  
**Models tested:** GPT-4o-mini, GPT-4o, Qwen2.5-1.5B/3B, Llama 3.2-1B/3B (+ DeepSeek-R1-32B, Claude 3.0/3.5 Haiku in appendix)

## 5.3. Notable Results

- A-MEM ranks **#1 on average across all six base models** on LoCoMo (F1 + BLEU-1), achieving consistent first-place ranking where no baseline does.
- **Multi-Hop tasks** show the largest gains: A-MEM achieves at least **2× the F1** of LoCoMo and MemGPT with GPT-4o-mini (27.02 vs. 25.02 / 26.65), and up to **6× ROUGE-L** gains for smaller models (Qwen2.5-1.5B: 27.23 vs. 4.68 for LoCoMo).
- On **DialSim**, A-MEM achieves F1 = 3.45 vs. LoCoMo's 2.55 (+35%) and MemGPT's 1.18 (+192%).
- **Token efficiency:** A-MEM uses ~1,200–2,500 tokens per query vs. ~16,900 for LoCoMo/MemGPT, an **85–93% reduction** translating to <$0.0003 per operation.
- **Scaling:** retrieval time grows from 0.31 µs (1K memories) to 3.70 µs (1M memories) — O(N) space, near-constant per-query cost, matching MemoryBank's footprint with richer representations.

---

# 6. Strengths

- **Backward-propagating evolution:** Unlike append-only memory systems, A-MEM retroactively enriches historical notes, allowing early memories to accumulate richer context as the agent learns.
- **Linked retrieval enhances long-range reasoning:** Returning an entire box on a single-note hit provides richer multi-hop context without increasing query cost, elegantly addressing long-range dependency tasks.
- **High token efficiency at no accuracy cost:** The selective top-k retrieval replaces full-context injection, cutting operational cost by an order of magnitude while consistently outperforming full-context baselines on complex reasoning categories.
- **Model-agnostic scalability:** Gains hold across a 1B–70B+ parameter range, suggesting the architecture imposes minimal capability requirements on the backbone LLM for the memory management operations themselves.

---

# 7. Gaps

- **Dependent on LLM backbone:** Memory quality is bounded by the backbone LLM; hallucinated keywords or spurious links propagate silently through evolution steps with no audit trail or confidence scoring, raising questions about long-term reliability as memory size grows.
- **Write cost under high-frequency interaction streams is unstudied:** Each memory ingestion requires multiple LLM calls (construction + link generation + evolution). The paper benchmarks retrieval scaling to 1M entries but does not analyze the compounding write cost under realistic agentic workloads with rapid consecutive interactions.
- **Evaluation is limited to conversational QA:** All benchmarks test information retrieval from dialogue. Open-ended agentic tasks (tool selection, multi-step planning, code execution loops) where memory must guide action rather than just answer questions, remain unexplored.
- **No forgetting or pruning policy:** The system grows monotonically. Whether link density and retrieval precision degrade gracefully as semantically redundant or outdated memories accumulate is not examined; this is especially relevant for deployed agents with indefinite lifespans.
- **Multimodal memory is architecturally unexplored:** The note schema is text-only. Extending to images, audio, or structured tool outputs would require non-trivial changes to embedding, LLM prompting, and link semantics

---
# 8. Highlights:

> [!PDF|] [[A-Mem.pdf#page=1&selection=36,42,37,7|A-Mem, p.1]]
> > basic principles of the Zettelkasten method,


> [!PDF|255, 208, 0] [[A-Mem.pdf#page=1&annotation=876R|A-Mem, p.1]]
> >  multiple structured attributes, including contextual descriptions, keywords, and tags
> 
> 


> [!PDF|255, 208, 0] [[A-Mem.pdf#page=1&annotation=887R|A-Mem, p.1]]
> >  While graph databases provide structured organization for memory systems, their reliance on predefined schemas and relationships fundamentally limits their adaptability.

> [!PDF|255, 208, 0] [[A-Mem.pdf#page=2&annotation=890R|A-Mem, p.2]]
> > The challenge becomes increasingly critical as LLM agents tackle more complex, open-ended tasks, where flexible knowledge organization and continuous adaptation are essential. 

> [!PDF|255, 208, 0] [[A-Mem.pdf#page=2&annotation=893R|A-Mem, p.2]]
> > enables autonomous generation of contextual descriptions, dynamic establishment of memory connections, and intelligent evolution of existing memories based on new experience
> 

> [!PDF|255, 208, 0] [[A-Mem.pdf#page=2&annotation=899R|A-Mem, p.2]]
> >  agentic memory update mechanism where new memories automatically trigger two key operations: link generation and memory evolution


> [!PDF|255, 208, 0] [[A-Mem.pdf#page=4&annotation=902R|A-Mem, p.4]]
> > Following the Zettelkasten principle of atomicity, each note captures a single, self-contained unit of knowledge.
> 
> 