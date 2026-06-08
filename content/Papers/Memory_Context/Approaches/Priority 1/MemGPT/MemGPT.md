---
Tags:
Date: "2024"
Authors: Charles Packer, Sarah Wooders, Kevin Lin, Vivian Fang, Shishir G. Patil, Ion Stoica, Joseph E. Gonzalez
Venue: Misc
Paper: "MemGPT: Towards LLMs as Operating Systems"
Memory type:
  - Token-level
Agent env: Single agent
Record format: Text
Memory architecture:
  - 2-tier
Tackle Module: Context optimization
Need offline initialization: false
Fine-tuning?: false
Other tags:
---
# 1. Terminology

| Term                    | Definition                                                                                                                                                                        |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Virtual Memory (OS)** | An OS abstraction that gives programs the illusion of more RAM than physically exists, by transparently paging data between RAM and disk. MemGPT borrows this idea.               |
| **Paging**              | Moving data blocks in/out of physical memory (RAM ↔ disk). MemGPT analogizes this as moving text in/out of the LLM's context window.                                              |
| **Main Context**        | Everything currently inside the LLM's context window (= RAM in OS terms). Directly accessible during inference.                                                                   |
| **External Context**    | Data stored outside the context window in databases (= disk in OS terms). Must be explicitly retrieved via function calls.                                                        |
| **Working Context**     | A fixed-size read/write region within the prompt tokens for storing persistent key facts (e.g., user's name, preferences). Survives across sessions.                              |
| **FIFO Queue**          | A rolling message buffer inside the context window. New messages are appended; old ones are evicted when space runs out (First-In, First-Out).                                    |
| **Archival Storage**    | A long-term, append-only external database for arbitrary-length data. Accessed via vector similarity search. Analogous to disk storage.                                           |
| **Recall Storage**      | An external message history database storing all past conversation turns. Supports keyword/semantic search. Analogous to a conversation log on disk.                              |
| **Function Calling**    | The ability of an LLM to emit structured outputs that trigger code execution (e.g., `archival_storage.search("six flags")`). The mechanism MemGPT uses for all memory operations. |
| **Function Chaining**   | Executing multiple sequential function calls before returning control to the user, enabled by the `request_heartbeat=true` flag. Allows multi-step retrieval.                     |
| **Queue Eviction**      | Removing old messages from the FIFO queue when the context is full. Evicted messages are summarized and archived, not discarded.                                                  |
| **Recursive Summary**   | A running compressed representation of evicted messages. Each new eviction appends to and re-compresses the existing summary.                                                     |
| **ROUGE-L (Recall)**    | A text overlap metric measuring how much of the reference answer appears in the generated answer. Used here to handle verbose model outputs.                                      |
| **Multi-hop Retrieval** | Answering a query that requires chaining multiple lookup steps (key → value₁ → value₂ → final answer), rather than a single retrieval.                                            |
| **HNSW Index**          | Hierarchical Navigable Small World: an approximate nearest-neighbor graph index enabling sub-second vector search over millions of embeddings. Used in MemGPT's archival storage. |
| **LLM-as-Judge**        | Using a capable LLM (e.g., GPT-4) to evaluate the quality of generated answers instead of (or alongside) string-matching metrics.                                                 |
| **MSC Dataset**         | Multi-Session Chat dataset (Xu et al., 2021) multi-turn dialogues where two users maintain consistent personas across 5 sessions. MemGPT's primary conversation benchmark.        |

---
# 2. Paper Summary (What)

MemGPT introduces a system that gives LLMs the **illusion of an infinite context window** by borrowing the concept of virtual memory from traditional operating systems. Instead of fitting everything into a fixed prompt, MemGPT treats the context window as scarce RAM and intelligently pages information in and out from external storage tiers using LLM-generated function calls.

---
# 3. What it Solves (Why)

Modern LLMs are capped by a fixed context window (e.g., 8k–128k tokens as of early 2024). Two hard consequences emerge from this:
- Chat agent forgets earlier sessions entirely.
- A document analyst cannot reason over files that exceed the window.
Truncation or chunking loses critical context.

---

## 4. Methodology (How)

MemGPT's architecture has three memory tiers and three control components.
> _a user named Alice who has multi-session conversations with a MemGPT agent._

---

### 4.1 Memory Hierarchy

![[MemGPT.pdf#page=3&rect=82,552,518,725|MemGPT, p.3]]
```
┌─────────────────────────────────────────────────────┐
│              LLM Finite Context Window               │
│  ┌──────────────┬──────────────┬───────────────────┐ │
│  │  System      │   Working    │    FIFO Queue      │ │
│  │  Instructions│   Context    │  (rolling msgs)    │ │
│  │  (read-only) │  (read/write)│  (read/write)      │ │
│  └──────────────┴──────────────┴───────────────────┘ │
└─────────────────────────────────────────────────────┘
          ▲ Retrieve                    ▲ Retrieve
          ▼ Write                       ▼ Write (auto)
┌──────────────────┐          ┌──────────────────────┐
│  Archival Storage│          │    Recall Storage     │
│  (long-term DB)  │          │  (message history DB) │
└──────────────────┘          └──────────────────────┘
```

- **System Instructions** (read-only): Static prompt defining MemGPT's behaviour, available functions, and memory rules.
> Alice's agent always knows _how_ to manage memory.

- **Working Context** (read/write, in-context):  Used to store key facts, preferences, and other important information about the user and the persona the agent is adopting.

> _After session 1, the agent writes: `"User name: Alice, Birthday: Feb 7, Partner: James"`_

- **FIFO Queue** (read/write, in-context): Recursive summary from evicted messages and rolling message history. When it fills up, older messages are evicted, summarized recursively, and pushed to Recall Storage.

> _By session 3, early messages about Alice's childhood are evicted. 
> A recursive summary captures the information: "Alice grew up in Chicago, loves Six Flags."_

- **Recall Storage** (external): Full message history, searchable by keyword or embedding similarity.

> _In session 5, Alice says "remember when we talked about Six Flags?" 
> => The agent calls `recall_storage.search("six flags")` and retrieves three past messages._

- **Archival Storage** (external): Arbitrary long-form data (documents, notes, embeddings). Append-only, vector-searchable.

> _In a document task, 20M Wikipedia embeddings live here. The agent pages through them 10 results at a time._

---

### 4.2 Queue Manager

Manages messages in FIFO queue and recall storage:

- **On new message**: append message to FIFO queue → trigger LLM inference → write both user message and LLM output to `Recall Storage`.
- **Memory pressure warning** (e.g., at 70% capacity): inject a System Alert: Memory Pressure message, giving the LLM a chance to proactively save critical facts to `Working Context or Archival Storage.`
- **Queue flush** (at 100% capacity): evict ~50% of oldest messages, generate a new recursive summary and store it in `FIFO 1st index`, also store evicted messages in `Recall Storage`.
- **Recall**: When messages from `recall storage` retrieved by LLM function call, it appends them back to FIFO queue to be included in the context again,

> _Alice is having her 6th long session. The queue fills. The agent sees the memory pressure warning and calls `working_context.append("Alice broke up with James on Feb 14")` before the flush evicts those messages._

---

### 4.3 Function Executor

LLM is provided with explicit instructions on how to interact with MemGPT memory:
- description of memory hierarchy and their utilities
- function schema to access and modify memory
The LLM's output is parsed as a function call. Key available functions include

| Function                                 | Purpose                                        |
| ---------------------------------------- | ---------------------------------------------- |
| `working_context.append(text)`           | Add a fact to the in-context scratchpad        |
| `working_context.replace(old, new)`      | Update a stored fact                           |
| `recall_storage.search(query)`           | Semantic search over message history           |
| `archival_storage.search(query, page=N)` | Paginated vector search over long-term storage |
| `archival_storage.insert(text)`          | Store new long-form data                       |

Function calls' results along with runtime errors (e.g., working context full) are fed back to the LLM so it can self-correct.

> _Alice mentions "I got a job at Google." The LLM calls `working_context.append("Works at Google")` and the next session greets her: "How's the new job at Google going?"_

---

### 4.4 Control Flow & Function Chaining

Events trigger inference: user messages, system alerts (memory pressure), scheduled tasks, or user ineractions (login, upload documents,...). 
The LLM can chain calls by emitting `request_heartbeat=true`, which immediately re-triggers inference after a function returns without user intervention.

> _For nested key-value lookup, the agent calls `archival_storage.search("key_A")` → gets `value_B` → immediately calls `archival_storage.search("value_B")` → gets `value_C` → … → returns final answer. All chained without the user seeing intermediate steps._

---

## 5. Benchmarks

### 5.1 Baselines

|Baseline|Description|
|---|---|
|**GPT-3.5 Turbo**|Fixed 16k context, no memory augmentation|
|**GPT-4**|Fixed 8k context, no memory augmentation|
|**GPT-4 Turbo**|Fixed 128k context, no memory augmentation|
|**Fixed-context + Recursive Summary**|Baseline receives a lossy summarization of prior sessions (mimics summarization pipelines)|
|**Fixed-context + Truncation**|Documents truncated to fit context; used for document QA beyond default limits|

---

### 5.2 Benchmark Tasks

| Task                            | Dataset                                              | What it Tests                                                                              |
| ------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Deep Memory Retrieval (DMR)** | Augmented [[Multi-Session Chat]]  (new session 6 QA) | Consistency: can the agent recall specific facts from sessions 1–5?                        |
| **Conversation Opener**         | MSC (session openers)                                | Engagement: does the agent craft personalized openers drawing on long-term user knowledge? |
| **Multi-Document QA**           | NaturalQuestions-Open + Wikipedia (50 questions)     | Document analysis across increasing numbers of retrieved documents (K = 1–200)             |
| **Nested Key-Value Retrieval**  | Synthetic UUID KV pairs (140 pairs, ~8k tokens)      | Multi-hop reasoning: follow chains of 0–4 nested lookups                                   |

---

### 5.3 Notable Results

**Deep Memory Retrieval (Accuracy / ROUGE-L Recall):**

|Model|Accuracy|ROUGE-L|
|---|---|---|
|GPT-3.5 Turbo|38.7%|0.394|
|**+ MemGPT**|**66.9%**|**0.629**|
|GPT-4|32.1%|0.296|
|**+ MemGPT**|**92.5%**|**0.814**|
|GPT-4 Turbo|35.3%|0.359|
|**+ MemGPT**|**93.4%**|**0.827**|

> GPT-4 + MemGPT nearly triples raw GPT-4's accuracy, despite GPT-4 having a much larger context than GPT-3.5.

**Conversation Opener (CSIM Similarity to Persona / Human):** MemGPT with all base models matches or exceeds human-written openers on SIM-1 and SIM-H scores 
![[MemGPT.pdf#page=5&rect=318,575,532,660&color=yellow|MemGPT, p.5]]
**Document QA:** Fixed-context baselines degrade monotonically as K grows due to truncation. MemGPT's accuracy remains flat and superior — it is not limited by context size since it actively pages through archival storage.
![[MemGPT.pdf#page=6&rect=55,577,291,727&color=yellow|MemGPT, p.6]]
**Nested KV Retrieval:** GPT-3.5 hits 0% accuracy at nesting level 1. GPT-4 and GPT-4 Turbo hit 0% by level 3. MemGPT with GPT-4 maintains high accuracy across all nesting levels (0–4), demonstrating reliable multi-hop reasoning.
![[MemGPT.pdf#page=7&rect=65,587,277,725&color=yellow|MemGPT, p.7]]

---

## 6. Strengths

- **Based on true OS memory design**: Paging, memory pressure warnings, eviction policies, and hierarchical tiers all have precise functional counterparts.

- **Model-agnostic design:** Experiments show consistent gains across GPT-3.5, GPT-4, and GPT-4 Turbo which indicates that the system is not tied to a single backbone.

- **Self-directed memory management.** Unlike RAG pipelines where retrieval is decided externally, MemGPT lets the LLM autonomously decide _what_ to store, _when_ to retrieve, and _how many_ hops to perform. This is a meaningful step toward genuine LLM agency.

- **Handles truly unbounded context.** Fixed-context and even long-context baselines have a hard ceiling. MemGPT's design is theoretically unbounded

- **Multi-hop reasoning capability.** The nested KV task demonstrates that function chaining enables compositional reasoning that flat-context models completely fail at beyond 2 hops.

---

## 7. Gaps

-  Flat FIFO queue causes topic mixing as conversation length grows 
- **Dependent on LLM backbone**: The quality of LLM summary, tool-calling and retrieval quality determine the performance of the system
- **Memory management policies (what to keep, what to evict, what to store) are entirely prompt-driven heuristics**. There is no learned policy that optimizes retrieval utility over time
	=> an RL or fine-tuning approach could substantially improve this.
- High toen cost for retrieval in long document analysis

---

## 8. Highlights

> [!PDF|] [[MemGPT.pdf#page=3&selection=93,0,96,1|MemGPT, p.3]]
> > MemGPT orchestrates data movement between main context and external context via function calls that are generated by the LLM processor.

> [!PDF|] [[MemGPT.pdf#page=2&selection=268,13,270,38|MemGPT, p.2]]
> > The first index in the FIFO queue stores a system message containing a recursive summary of messages that have been evicted from the queue.


> [!PDF|] [[MemGPT.pdf#page=2&selection=260,0,264,53|MemGPT, p.2]]
> > In conversational settings, working context is intended to be used to store key facts, preferences, and other important information about the user and the persona the agent is adopting, allowing the agent to converse fluently with the user

> [!PDF|] [[MemGPT.pdf#page=2&selection=265,0,265,47|MemGPT, p.2]]
> > FIFO queue stores a rolling history of messages

> [!PDF|] [[MemGPT.pdf#page=1&selection=124,2,126,44|MemGPT, p.1]]
> > Using function calls, LLM agents can read and write to external data sources, modify their own context, and choose when to return responses to the user.

> [!PDF|] [[MemGPT.pdf#page=1&selection=46,20,50,1|MemGPT, p.1]]
> > a system that intelligently manages different storage tiers in order to effectively provide extended context within the LLM’s limited context window.

> [!PDF|] [[MemGPT.pdf#page=6&selection=210,0,213,1|MemGPT, p.6]]
> > In this task, a question is selected from the NaturalQuestions-Open dataset, and a retriever selects relevant Wikipedia documents for the question.
> 
> Multi document question answering

> [!PDF|] [[MemGPT.pdf#page=8&selection=2,0,3,35|MemGPT, p.8]]
> > where values themselves may be keys, thus requiring the agent to perform a multi-hop lookup
> 
> Nested key value retrieval

