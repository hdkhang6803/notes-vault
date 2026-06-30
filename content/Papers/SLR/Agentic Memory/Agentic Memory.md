---
Tags:
Date: "2026"
Authors: Dongming Jiang, Yi Li, Songtao Wei, Jinxin Yang, Ayushi Kishore , Alysa Zhao , Dingyi Kang, Xu Hu, Feng Chen, Qiannan Li and Bingzhe Li
Venue:
Paper: "Anatomy of Agentic Memory: Taxonomy and Empirical Analysis of Evaluation and System Limitations"
---
# 1. Terminology

| **Term**                              | Definition                                                                                                                                       |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Memory-Augmented Generation (MAG)** | Paradigm extending LLM agent memory beyond the context window via external, non-parametric memory stores that evolve across interactions         |
| **Agentic Memory**                    | An external subsystem that influences agent behaviour through explicit read–write operations rather than weight updates                          |
| **Inference-time Recall**             | The process of querying and retrieving relevant memory entries to condition the agent's decision at each step                                    |
| **Memory Update**                     | Write operations (STORE, UPDATE, SUMMARIZE, LINK, EVICT, DELETE) that maintain a useful long-term memory state                                   |
| **Context Saturation Gap (Δ)**        | Performance difference between a MAG system and a brute-force Full-Context baseline; used as a diagnostic for whether external memory adds value |
| **Backbone Sensitivity**              | Variation in MAG system accuracy and format compliance depending on the underlying LLM (e.g., GPT-4o-mini vs. Qwen-2.5-3B)                       |
| **Silent Failure**                    | Memory corruption caused by a weak backbone producing malformed structured outputs during write operations, without surfacing an overt error     |
| **Agency Tax**                        | The latency and token overhead introduced by memory maintenance operations (extraction, update, consolidation) in agentic systems                |
| **Benchmark Saturation**              | Condition where all task-relevant information fits within a single long-context prompt, making external memory unnecessary                       |
| **LLM-as-a-Judge**                    | Evaluation protocol using an LLM (e.g., GPT-4o-mini) as a proxy for human semantic judgment, in place of lexical metrics                         |
| **Paraphrase Penalty**                | Lexical metric failure mode: correct abstractive answers penalised for low token overlap with the gold answer                                    |
| **Negation Trap**                     | Lexical metric failure mode: high token overlap masking a factually inverted (negated) answer                                                    |
| **Top-k Recall**                      | Standard retrieval: select k memory entries maximising a scoring function (dense similarity, sparse matching, or reranking)                      |
| **Utility-aware Retrieval**           | Idealised retrieval selecting memory entries that maximise downstream agent utility rather than semantic similarity alone                        |

---

# 2. Review Protocol

This paper is<span style="color:rgb(192, 0, 0)"> a structured survey combined with original empirical analysis, not a classical PRISMA-style SLR</span>. The review protocol operates on two tracks:

- **Architectural track.** The authors survey recent MAG literature and organise it into a four-category structural taxonomy (see 3). Systems were selected based on representativeness across the four categories; no explicit inclusion/exclusion count is reported.

- **Empirical track.** Six MAG architectures are selected for head-to-head evaluation: AMem, MemoryOS, Nemori, MAGMA, SimpleMEM, and MemSkill. Evaluations run on the LoCoMo benchmark using two backbone models (GPT-4o-mini and Qwen-2.5-3B). All systems use a standardised embedding model (all-MiniLM-L6-v2), temperature 0.3, and top-k = 10 for final answer retrieval. Evaluation covers four dimensions: 
	- benchmark saturation risk, 
	- metric validity (F1 vs. LLM-as-a-judge), 
	- backbone sensitivity, 
	- system latency/cost. 
	The authors compare their survey's coverage against six concurrent surveys via a feature-presence table (Table 1).

---

# 3. Categories

The paper proposes a four-category structural taxonomy of MAG systems:

- **Lightweight Semantic Memory**: memory as independent textual units in a vector space, retrieved via top-k similarity. Subdivided into: 
	- **RL-Optimised Semantic Compression** (e.g., MemAgent, MemSearcher): Treat memory as fixed-size store and apply RL to optimize how data is retained and overwritten.
	- **Heuristic/Prompt-Optimised** (e.g., ACON, CISM, SimpleMem): Prompt design or heuristically summarize prior steps as flat textual representation.
	- **Context Window Management** (e.g., AgentFold, Context-Folding Agent): Reorganized information to fit a bounded window without accumulating memories (Focus on local reasoning > long-term memory)
	- **Token-Level Semantic Memory** (e.g., MemGen, TokMem): Using dedicated memory tokens or compressed latent panel

- **Entity-Centric and Personalized Memory**: memory organized around explicit entities (users, tasks, preferences) using structured attribute-value records. Subdivided into:
	- **Entity-Centric Memory** (e.g., A-MEM, Memory-R1, Mem0): Memory represented á entitites (notes,...)
	- **Personalized Memory** (e.g., PAMU, EgoMem, MemOrb, MemoryBank): Persistent user profile, preference to support  identity -consistent behavior across sessions

- **Episodic and Reflective Memory**: temporal abstraction via episodes or higher-level summaries, with periodic consolidation. Subdivided into: 
	- **Episodic Buffer with Learned Control** (e.g., MemR3): episodic interaction in bounded buffer and looping through insert, retain or delete through learned policies
	- **Episodic Reflection & Consolidation** (e.g., MemP, LEGOMem, TiMem, Nemori): Reflect and consolidate memory in a compact representation (abstrraction, hierarchical layers)
	- **Episodic Recall for Exploration** (e.g., EMU, SAM2RL): Use episodic memory for exploration.
	- **Episodic Utility Learning** (e.g., MemRL, Memory-T1): Memory is augmented with evolving learned signals to selectively retain and retrieve records

- **Structured and Hierarchical Memory** :explicit relational or multi-tier organisation. Subdivided into: 
	- **Graph-Structured Memory** (e.g., MAGMA, Zep, SGMem, LatentGraphMem): Organise memory as nodes and edges
	- **OS-Inspired & Hierarchical Memory** (e.g., MemGPT, MemoryOS, EverMemOS, HiMem): Organize memory into tiers (long-term, short-term, working)
	- **Policy-Optimised Memory Management** (e.g., MEM1, Mem-α, AtomMem): Treat memory operations as learnable decisions via RL or hybrid training.

---

# 4. Gaps

- **Benchmark saturation (Table 2).** Most existing benchmarks fall within the reach of long-context LLMs -> Do not need external memory: (Total token load + Interaction depth + entity diversity)
	- HotpotQA (~1k tokens, single-turn) and MemBench (~100k tokens) both fit inside a 128k context window and carry high saturation risk. 
	- LoCoMo (~20k tokens, 35 sessions) requires multi-session reasoning but remains in-window. 
	- LongMemEval-M (>1M tokens) is the only tested dataset that structurally necessitates external memory.
	=> Propose **Context Saturation Gap** (Performance Gain before and after memory augmentation)

- **Metric validity (Table 3).** F1 and LLM-as-a-judge rankings diverge substantially.
	- MAGMA ranks 1st by semantic judge (Prompt 1: 0.670) but 2nd by F1 (0.467). 
	- AMem ranks 4th semantically but 5th by F1 (0.116) - penalised for abstractive paraphrase. 
	- SimpleMem receives a higher F1 (0.268) than its semantic score warrants (<0.30).
	=> Paraphrase Penalty & Negation Trap -> Optimize F1 will favor surface memorization instead of reasoning and memory integration
	=> Semantic judge rankings remain consistent across three distinct prompt sources (MAGMA, Nemori, SimpleMem), demonstrating ranking robustness.

- **Backbone sensitivity (Table 4).** Switching from GPT-4o-mini to Qwen-2.5-3B causes
	- Nemori's answer score to drop from 0.781 to 0.447 and its format error rate to rise from 17.91% to 30.38%
	- SimpleMem drops from 0.289 to 0.102 with format errors rising from 1.20% to 4.82%
	=> Structured architectures (graph-based, episodic) degrade more severely than append-only systems under weaker backbones.

- **System latency and cost (Table 5).** 
	- MemoryOS is the clear latency outlier at 32.37s per turn, making it impractical for interactive use. 
	- SimpleMem (1.06s), Nemori (1.13s), and LOCOMO (0.78s) achieve efficient user-facing latency. 
	- Full Context incurs the highest generation latency (1.73s) despite no retrieval step.
	- AMem requires ~15 hours for offline index construction. 
	- Nemori consumes the most tokens during construction (7.04M), approximately 5× SimpleMem (1.3M). 
	- MAGMA offers a Pareto-favourable balance (~1.46s latency, 2.7M construction tokens).
	=> Structured memory needs robust asynchronous infrastructure

---

# 5. Findings

- The Context Saturation Gap (Δ = Score_MAG − Score_FullContext) should be reported as a diagnostic in all MAG evaluations, rather than measuring MAG performance in isolation.
- Lexical metrics (F1) systematically diverge from semantic judgments due to four recurring failure modes: Surface Variation, Semantic Equivalence Gap, Polarity Flip, and Entity Drift.
- LLM-as-a-judge produces stable architecture rankings across different rubrics and is a more reliable protocol than F1 for agentic memory evaluation, though absolute scores vary with prompt strictness.
- Weaker open-weight backbones (e.g., Qwen-2.5-3B) cause silent memory corruption via malformed structured outputs during write operations — a failure mode invisible at the surface but destructive to long-term memory integrity.
- Graph-based and episodic architectures are disproportionately vulnerable to backbone degradation due to their reliance on structured generation (entity extraction, relation construction, deduplication).
- Memory maintenance overhead is a hidden scalability bottleneck: if asynchronous update pipelines lag behind user interactions, memory becomes stale and system quality degrades.
- There is no single best MAG architecture; the right choice depends on the trade-off between reasoning quality (structured/hierarchical > lightweight), latency (lightweight > structured), cost (lightweight > episodic/graph), and backbone robustness (append-only > structured).

---

# 6. Memory/Context Related

**Memory architectures surveyed.** The paper covers the full MAG landscape across four structural categories with ~50+ systems catalogued. Key systems include: MemGPT (LLM-driven memory paging), MAGMA (multi-relational graph with semantic/temporal/causal/entity layers), Nemori (episodic tree + semantic graph with cognitive-science-inspired boundary detection), MemoryOS (three-tier STM→LTM hierarchy), A-MEM (linked knowledge notes with LLM-generated edges), MEM1 (constant-memory RL policy), Mem-α (multi-component external memory under ultra-long contexts), TiMem (temporal-hierarchical memory tree without RL/fine-tuning), and MemSkill (self-evolving memory skills).

**Memory operations formalised.** The paper provides a formal model: query generation qt = Query(ot, st); retrieval rt = Read(Mt, qt); action generation at ~ πθ(ot, rt, st); memory update Mt+1 = Write(Mt, ot, at, st). Memory actions are typed as STORE, UPDATE, SUMMARIZE, LINK, EVICT, or DELETE. Ideal retrieval is framed as utility maximisation r*t = argmax E[U(at | ot, r, st)] rather than pure similarity.

**Benchmarks evaluated for saturation risk.**

|Benchmark|Volume|Interaction Depth|Entity Diversity|Saturation Risk|
|---|---|---|---|---|
|HotpotQA|~1k tokens|Single turn|Low|High (trivial for context window)|
|LoCoMo|~20k tokens|35 sessions|High|Moderate (requires reasoning)|
|LongMemEval-S|103k tokens|5 core abilities|High|Moderate (borderline)|
|LongMemEval-M|>1M tokens|5 core abilities|High|Low (requires external memory)|
|MemBench|~100k tokens|Fact/reflection|Medium|High (fits in 128k window)|

**Context window management.** The paper identifies context window management as a distinct MAG sub-category (within Lightweight Semantic Memory), covering systems that fold, summarise, or reorganise prior interactions within a bounded window for local reasoning efficiency without cross-session persistence. Examples: AgentFold (multi-scale folding via learned operations), Context-Folding Agent (RL-based sub-task branching with segment compression).

**Key limitations of current memory systems.**

- Benchmark saturation: most benchmarks do not structurally require external memory given 128k+ context windows.
- Metric misalignment: F1 fails to capture abstractive correctness; LLM-as-a-judge requires careful prompt calibration.
- Backbone dependence: structured memory operations (JSON generation, entity extraction, graph construction) degrade significantly under smaller or open-weight models.
- System cost: maintenance overhead (write latency, consolidation tokens) is rarely measured but can collapse throughput at scale.
- Survey scope: the taxonomy may miss very recent or concurrent systems given the rapid pace of the field.

---

# 7. Future directions
- Future benchmarks should be saturation-aware and move beyond lexical overlaps
- Build agentic memory balanced between accuracy, latency, cost and reliability


# Highlights