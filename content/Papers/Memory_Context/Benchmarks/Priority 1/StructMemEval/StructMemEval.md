---
Tags:
Date: February 10, 2026
Authors: Yiming Xiong, Shengran Hu, Jeff Clune (University of British Columbia / Vector Institute)
Venue:
Paper:
---
# 1. Terminology

| **Term**                      | Definition                                                                                                                                    |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Long-term memory**          | External database allowing an LLM agent to offload knowledge from its working memory (KV cache) and retrieve it on demand                     |
| **Working memory / KV cache** | Token-level key-value representations held in accelerator memory during inference; limited to ~10⁴–10⁵ tokens in most models                  |
| **Retrieval-augmented LLM**   | A baseline where the full conversation is embedded and top-k chunks are retrieved verbatim to answer a query; no memory transformation occurs |
| **Memory agent**              | An LLM equipped with tools to actively create, update, and query an external memory store (e.g., Mem0, Mem-agent)                             |
| **Memory organization hint**  | An optional natural-language prompt that tells the agent which data structure to use when maintaining its memory for a given task             |
| **Netting**                   | Canceling circular debts among multiple parties to compute the minimal final settlement amounts                                               |
| **State transition**          | An event in a conversation that invalidates or modifies a previously held fact (e.g., a user moves cities, a KANBAN card changes status)      |
| **LLM-as-a-judge**            | Using a separate LLM call to evaluate free-form answers that cannot be checked by exact match                                                 |

---

# 2. Metadata

| Property          | Value                                                                                                                                                                                                                               |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Size**          | 73 unique scenarios (conversations); 544 evaluation questions total                                                                                                                                                                 |
| **Splits**        | Tree-based, State tracking, Count-based (+ planned: ordered lists, DAGs, assignment maps, indexes)                                                                                                                                  |
| **Composition**   | Tree-based: family trees & corporate hierarchies, 10–250 edges<br>State tracking: 0–5 relevant transitions, 267 sessions / 1,453 messages<br>Count-based: multi-party transaction ledgers, 10/20/50 transactions, 10%/30%/50% noise |
| **Input format**  | Sequential conversation messages fed one-at-a-time to the memory system; each message encodes a relation, event, or transaction in natural language                                                                                 |
| **Output format** | Closed-form answers (yes/no, numeric amounts, named path) evaluated by exact match; open-form answers evaluated by LLM-as-a-judge                                                                                                   |
| **Availability**  | Public: https://github.com/yandex-research/StructMemEval                                                                                                                                                                            |
| **Data origin**   | Hybrid: human-authored seed scenarios → LLM augmentation (DeepSeek-v3, Qwen3-max, Claude Opus 4.5) → manual verification                                                                                                            |

---

# 3. What it Measures (What)

## 3.1. Task Definition

StructMemEval tests whether an LLM memory agent can **organize** its long-term memory into task-appropriate structures, not merely retrieve stored facts. Tasks are evaluated in two modes: **without hints** (main setting) and **with hints** (diagnostic setting to isolate organizational failure from execution failure).

Three task families are included:
- **Tree-based:** The agent receives a stream of relational statements (e.g., "A is B's stepdaughter") and must maintain a complete family tree or corporate hierarchy. Queries ask for indirect relations that require traversing the graph, deliberately confusing pure retrieval.
- **State tracking:** Entities transition through states over time (e.g., a person moves cities). The agent must track the _current_ state rather than the most frequently mentioned one. Difficulty is parametrized by the number of state transitions (0–5).
- **Count-based:** The agent observes a history of financial transactions among multiple parties and must compute the final net settlement. Difficulty scales with the number of transactions (10/20/50) and the fraction of noise messages (10%/30%/50%).

## 3.2. Example Records

- **Tree-based**:
	- Input stream: _"Carol is David's wife"_, _"Emma is David's stepdaughter"_ … 
	- Query: _"Is Emma related to Carol, and if so, how?"_ (requires inferring Carol as Emma's stepparent)

- **State tracking**:
	- Input stream: _"I owe $200 to my neighbor Alice"_, _"I moved to City Y"_ … 
	- Query: _"Do I still owe Alice anything?"_ (correct answer: no, because the neighbor relationship no longer holds after the move)

- **Count-based**:
	- Input stream: _"Alice paid $120 for train tickets for everyone"_, _"Bob paid $57 for lunch for everyone"_ … 
	- Query: _"What is the final settlement between Alice and Bob?"_ (requires summing and netting all transactions)

## 3.3. Metrics

|Metric|Applied to|
|---|---|
|**Exact match accuracy**|Yes/no questions (tree relations), numeric settlement amounts, explicit path answers|
|**LLM-as-a-judge**|Questions without a canonical surface form|
|**Per-difficulty accuracy curves**|Accuracy plotted against number of edges / transitions / transactions to reveal scaling behavior|

---

# 4. Uniqueness (Why)

## 4.1. Other Benchmarks

| Benchmark                              | Focus                                                                                                          | Limitation noted by authors                                                                                                      |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **[[LoCoMo]]** (Maharana et al., 2024) | Long-term conversational recall in a personal assistant setting; single-hop, multi-hop, time-sensitive updates | Simple retrieval baselines (EMem) outperform complex memory architectures, suggesting the tasks don't stress memory organization |
| **[[LongMemEval]]** (Wu et al., 2024)  | Chat assistant recall over extended interactions                                                               | ^                                                                                                                                |
| **[[MemoryBench]]** (Ai et al., 2025)  | Memory and continual learning                                                                                  | Primarily targets factual retention                                                                                              |
| **[[StoryBench]]** (Wan & Ma, 2025)    | Multi-turn narrative memory                                                                                    | Conversational, not structured-organization focused                                                                              |
| **[[Evo-Memory]]** (Wei et al., 2025)  | Test-time self-evolving memory                                                                                 | Targets adaptation, not structural organization                                                                                  |

## 4.2. Uniqueness

StructMemEval is the first benchmark to explicitly target **memory organization capability** as a distinct competency, decoupled from factual recall, reasoning difficulty, or domain knowledge. Its key differentiating design choices are:

- **Structure-first task design:** Every task is chosen because a _notepad-augmented human_ would naturally reach for a specific data structure (tree, ledger, state machine). Tasks are trivial with the right structure and nearly unsolvable without it.
- **Implementation-agnostic evaluation:** Only final answers are judged, not internal memory representations, making it applicable to graph-based (Mem0/Zep), note-based (A-Mem, Mem-agent), and any future memory framework.
- **Diagnostic hint mechanism:** The with/without-hint split allows precise attribution of errors (if a system fails without hints but succeeds with them, the failure is organizational; if it fails even with hints, the failure is executional (maintenance or retrieval)).
- **Retrieval as a deliberately weak baseline:** Unlike benchmarks where retrieval matches or beats agentic memory, StructMemEval is specifically engineered so that retrieval degrades gracefully as task complexity grows, validating that its tasks genuinely require structural memory.

---

# 5. Generation Method (How)

All three task families follow a shared three-stage pipeline: 
1. human annotators manually author seed scenarios to validate feasibility
2. LLM augmentation generates additional scenarios and evaluation questions Different LLMs per subset to reduce style bias: 
	- DeepSeek-v3 for accounting and tree tasks
	- Qwen3-max for recommendation/count tasks
	- Claude Opus 4.5 for state tracking
3. human verifiers correct hallucinations and validate reference answers.

Task-specific details:
- **Count-based:** Seed transactions authored manually; DeepSeek generates similar ones plus noise messages at three noise levels (10%/30%/50%). Reference settlements computed by an LLM with arithmetic tools, then manually corrected.
- **Tree-based:** A reference family-tree graph is generated first; scenarios are created as subsets of 10 to 250 edges, all guaranteed to include a 10-hop query path. DeepSeek (with tool use) augments relation messages.
- **State tracking:** Seed dialogues with 0–5 transitions authored manually; Claude Opus 4.5 augments them. Each difficulty level uses distinct conversations rather than extending earlier ones.

Three different LLMs are used deliberately across subsets to reduce the risk of implicit style contamination that could favor any one evaluation model.

---

# 6. Gaps

- **Coverage of structure types is narrow:** Only trees, state machines, and ledgers are currently included. The authors acknowledge the absence of ordered lists (to-do queues, top-K ranking), DAGs, assignment maps (calendars, resource allocation), and multi-structure tasks
- **Single modality and domain:** All tasks are text-based and synthetic. Generalization to real-world deployments (user-facing assistants, code agents, embodied agents) is unvalidated.
- **Evaluation infrastructure dependency:** The retrieval baseline relies on proprietary OpenAI embeddings, which may disadvantage open-source reproducibility and limit fair comparison against systems using different embedding models.

---
# 7. Highlights

> [!PDF|255, 208, 0] [[StrucMemEval.pdf#page=1&annotation=845R|StrucMemEval, p.1]]
> >  Most longterm memory benchmarks focus on simple fact retention, multi-hop recall, and time-based changes

> [!PDF|255, 208, 0] [[StrucMemEval.pdf#page=1&annotation=848R|StrucMemEval, p.1]]
> > a benchmark that tests the agent’s ability to organize its long-term memory, not just factual recall.




