---
Tags:
Date: February 10, 2026
Authors: Yiming Xiong, Shengran Hu, Jeff Clune (University of British Columbia / Vector Institute)
Venue: NeurIPS
Paper: "G-Memory: Tracing Hierarchical Memory for Multi-Agent Systems"
---
| Term                                           | Definition                                                                                                                                                                                                         |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Organizational Memory Theory**               | A management-science concept describing how organizations store, retrieve, and use collective knowledge. G-Memory borrows this idea to give AI agent teams a shared institutional memory.                          |
| **Inside-trial Memory**                        | Information an agent remembers only within one task session like short-term memory. Once the task is done, it is gone. Example: an agent remembering mid-conversation context.                                     |
| **Cross-trial Memory**                         | Experience that persists across multiple separate tasks like long-term memory. Example: remembering that a previous similar task failed due to a specific mistake.                                                 |
| **Graph Sparsifier**                           | An algorithm that takes a large, dense graph and removes less important nodes/edges to produce a smaller, informative summary. In G-Memory, this compresses lengthy agent dialogues into key steps.                |
| **Bi-directional Memory Traversal**            | G-Memory's retrieval strategy that searches both upward (query → insight graph) for abstract lessons and downward (query → interaction graph) for detailed procedural steps.                                       |
| **Hop Expansion**                              | A graph search technique that broadens a query result by also including graph neighbors of the initial matches (e.g., 1-hop = direct neighbors). Used to find semantically related but not identical past queries. |
| **Interaction Graph (Utterance Graph)**        | The lowest-level graph in G-Memory storing fine-grained text exchanges between agents during a task — like a detailed meeting transcript with causal links between messages.                                       |
| **Query Graph**                                | The mid-level graph storing metadata about past task queries: the task itself, whether it succeeded or failed, and links to related queries and interaction graphs.                                                |
| **Insight Graph**                              | The highest-level graph storing abstract, generalizable lessons distilled from multiple tasks. Example insight: _"Always verify object states before and after an action."_                                        |
| **Plug-and-Play Module**                       | A component that can be added to an existing system without modifying its internal structure. G-Memory can be bolted onto AutoGen, DyLAN, or MacNet without rewriting those frameworks.                            |
| **SOP (Standard Operating Procedure)**         | Pre-defined workflows that dictate how agents interact. Many early MAS (e.g., MetaGPT, ChatDev) rely on SOPs, limiting their ability to adapt from experience.                                                     |
| **Embodied Action Task**                       | A task where an agent must interact with a simulated physical environment (e.g., navigate a room, pick up and clean an object). Benchmarks: ALFWorld, SciWorld.                                                    |
| **PDDL (Planning Domain Definition Language)** | A formal language for describing planning problems (e.g., stacking blocks in a specific order). Used as a game benchmark in the paper to test strategic reasoning in MAS.                                          |
# 2. Paper Summary (What)
G-Memory (**G**raph-based **A**gentic **M**emory **M**echanism) is a hierarchical, plug-and-play memory system designed specifically for LLM-based Multi-Agent Systems. G-Memory organizes a team's collective experience into a **three-tier graph hierarchy**:
- **Insight Graph**: Abstract, generalizable lessons distilled from multiple past tasks (e.g., _"Always verify object state before acting"_).
- **Query Graph**: A network of past task queries annotated with success/failure status and semantic links to related queries.
- **Interaction Graph**: Fine-grained records of what each agent said during a task, linked causally (agent A's message inspired agent B's response).
# 3. What it solves (Why)
- Multi-agent System lack cross-trial memory (mostly inter-trial or overly condensed artifacts) => Need memory for MAS
- MAS has much longer trajectories and interaction history so can not scale up single-agent memory easily => Need organized hierarchical memory
- Each agent needs its own piece of memory => Need customization for each agent
# 4. Methodology (How)

## 4.1. Three-tier graph architecture

- **Interaction graph**:
	- nodes as the agents and textual content
	- edges as temporal relationship
- **Query graph**:
	- nodes as the query, status and its interaction graph
	- edges as the semantic relationship
- **Insight graph**:
	- nodes as distilled insights and supporting queries
	- edges as hyper-connections of insights contextualizing each other through a specific query
>For example:
>- **Insight Graph** = The firm's general legal principles and best practices (_"Never admit liability before consulting the client"_)
>- **Query Graph** = An index of past cases: outcomes, case types, and cross-references to similar precedents
>- **Interaction Graph** = The full meeting transcripts and argument chains from each case

## 4.2. Memory retrieval
### 4.2.1. Coarse-grained Similarity Search
G-Memory embeds the new query and all past queries as numerical vectors (using MiniLM).
It retrieves the top-_k_ most semantically similar past queries from the Query Graph using cosine similarity
> **Example: ALFWorld Task Retrieval**
> - New Task: `{"put a clean cloth on the countertop"}`
> - G-Memory searches the Query Graph and finds: `{"put a clean egg in the microwave"}`
> - Key shared feature: both tasks require the object to be in a **clean state** before placement.

### 4.2.2. Hop Expansion
The initial similarity search may miss indirectly related queries => G-Memory expands the candidate set by including **1-hop neighbors** (directly connected nodes in the Query Graph)
> **Example Hop Expansion:**
> - Initial match: `{"put a clean egg in microwave"}`
> - 1-hop neighbors in Query Graph: `{"clean a knife and place it in the drawer"}`, `{"heat an apple and put it on a plate"}`
> - These neighbors were linked because they share the procedural pattern: _clean/heat → then place_, and G-Memory pulls them in to enrich the context.

### 4.2.3. Bi-directional Memory Traversal

With the expanded query candidate set, G-Memory traverses in both directions:
- **Upward (Query -> Insight Graph):** Retrieves high-level insights associated with the matched queries (abstract lessons applicable to the whole team).
- **Downward (Query -> Interaction Graph):** Retrieves the most relevant past interaction logs. An LLM-based **graph sparsifier** then prunes these logs, keeping only critical steps and removing irrelevant dialogue to avoid context overload.
>**Example: Upward + Downward Retrieval (HotpotQA):**
>- Task:  `{"Are Deodato and Alejandro both film directors?"}`
>- **Upward:** Insight retrieved → `{"Verify that search results are not mistakenly referring to similar entities with similar names."}`
>- **Downward:** A compressed trajectory is retrieved showing a past agent team successfully identifying a film director by explicitly searching their filmography, not just their Wikipedia biography.

### 4.2.4. Role-specific Memory Injection
G-Memory does **not** give the same memory to every agent.
A filtering function evaluates each retrieved insight and interaction snippet against each agent's specific role and the current task:

## 4.3. Memory update
Once the MAS completes the task and receives environment feedback (success/failure, token count, etc.), G-Memory updates all three graph levels:

1. **Interaction Level:** The new task's full agent dialogue is traced and stored as a new Interaction Graph node.
2. **Query Level:** A new query node is added to the Query Graph, linked to the retrieved historical queries and to the insights that guided the task.
3. **Insight Level:** New insights are distilled by an LLM comparing success and failure trajectories. Existing insights are updated to reflect their relevance to the new task. Insights that prove consistently useful across many tasks become more strongly connected in the graph.
# 5. Strengths
- **Strong empirical coverage:** 5 benchmarks × 3 LLMs × 3 MAS frameworks is a rigorous and uncommon evaluation breadth for a memory paper.
- **Principled design grounded in theory:** Organizational Memory Theory provides a sound conceptual anchor distinguishing G-Memory from ad hoc engineering solutions.
- **Token efficiency:** Achieving state-of-the-art performance without proportionally increasing token cost is practically important given LLM API costs.
- **Plug-and-play design:** Integrating into existing frameworks without modifying their code lowers the barrier to adoption.
- **Role-specific memory:** Recognizing that different agents need different memory is a nuanced insight that most prior work ignores.
- **Ablation and sensitivity studies:** The paper carefully validates each component (insights vs. interactions) and tunes hyperparameters (hop count, query count), adding scientific rigor.
# 6. Gaps
- **Limited Domain Coverage:** Only benchmark on 3 domains
	=> Expand the benchmark
- **Scalability of the Memory Graph:** The paper does not address how retrieval latency and relevance quality degrade as graph size increases, nor does it propose graph pruning strategies to remove outdated or contradicted insights over time.
	=> Forgetting mechanism and compaction strategies
- **Depends on FM capacity:** The insight summarization and the graph sparsifier are LLM-powered, meaning they are susceptible to generating plausible-sounding but incorrect insights
- **Fixed Memory Retrieve and Update Moment:** Only triggered at the start and after a query -> May lose mid-task insights
	=> The paper addressed this as a customizable config

# 7. Benchmarks
## 7.1. Other baselines:
- [[Voyager]]
- [[Memory Bank]]
- [[Generative Agent]]
- [[MetaGPT]] 
- [[ChatDev]] 
- [[MacNet]]
## 7.2. Benchmarks:
- [[ALFWorld]]: Embodied action
- [[SciWorld]]: Embodied action
- [[PDDL]]: Game
- [[HotpotQA]]: Knowledge reasoning
- [[FEVER]]: Knowledge reasoning

## 7.3. MAS Frameworks:
- [[AutoGen]]
- [[DyLAN]] 
- [[MacNet]]

| Benchmark             | **MacNet (Best Baseline)** | **G-Memory** | **Improvement** |
| --------------------- | -------------------------- | ------------ | --------------- |
| ALFWorld              | 76.55                      | **88.81**    | **+12.26%**     |
| SciWorld              | 55.44                      | **67.40**    | **+11.96%**     |
| PDDL                  | 23.94                      | **27.77**    | +3.83%          |
| HotpotQA              | 28.36                      | **35.67**    | **+7.31%**      |
| FETCH (5th benchmark) | 60.87                      | **66.24**    | +5.37%          |
| **Average**           | 48.83                      | **57.18**    | **+8.35%**      |

- **G-Memory overhead:** 5.8M tokens
- **Others second-best overhead:** > 6M tokens
- **Implication:** G-Memory achieves better performance (especially on ALFWorld) while using significantly less memory/tokens than the competing MetaGPT-M baseline
![[G Memory Tracing Hierarchical Memory for Multi-Agent Systems.pdf#page=8&rect=107,556,240,723|G Memory Tracing Hierarchical Memory for Multi-Agent Systems, p.8]]
# 8. Highlights:

> [!PDF|255, 208, 0] [[G Memory Tracing Hierarchical Memory for Multi-Agent Systems.pdf#page=4&annotation=1663R|G Memory Tracing Hierarchical Memory for Multi-Agent Systems, p.4]]
> > majority voting schemes [47 ], hierarchical summarization via dedicated aggregator agents [13 , 30 ], or simply adopting the final agent’s output as the answer [ 46].
> 
> Aggregation operator to generate the final answer


> [!PDF|yellow] [[G Memory Tracing Hierarchical Memory for Multi-Agent Systems.pdf#page=8&selection=99,52,112,61&color=yellow|G Memory Tracing Hierarchical Memory for Multi-Agent Systems, p.8]]
> > Voyager and MemoryBank degrade AutoGen’s performance on PDDL by as much as 4.17% and 1.34%, respectively. We attribute this to the inability of these methods to provide agent role-specific memory suppor
> 
> 




