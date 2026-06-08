---
Tags:
Date: "2025"
Authors: Jiazheng Kang, Mingming Ji, Zhe Zhao, Ting Bai
Venue: EMNLP
Paper: Memory OS of AI Agent
Memory type:
  - Token-level
Agent env: Single agent
Record format: Text
Memory architecture:
  - 3-tier
  - graph-1
Tackle Module: Memory management
Need offline initialization: false
Fine-tuning?: false
Other tags:
  - Heat-based
---
# 1. Terminology

| Term                                | Definition                                                                                                                               |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Dialogue Page**                   | The atomic memory unit: a single turn structured as {Query, Response, Timestamp}                                                         |
| **STM** (Short-Term Memory)         | A fixed-length FIFO queue storing recent dialogue pages, each as (Q, R, T) tuples with chain metadata                                    |
| **Dialogue Chain**                  | Metadata linking semantically continuous pages within STM for coherent short-horizon context                                             |
| **MTM** (Mid-Term Memory)           | A segmented-paging store grouping topically related dialogue pages into segments                                                         |
| **Segment**                         | A topic-coherent group of dialogue pages in MTM, analogous to a logical OS memory segment                                                |
| **LPM** (Long-term Personal Memory) | Persistent storage for User Persona (profile, KB, traits) and Agent Persona (profile, traits)                                            |
| **Heat Score**                      | A composite relevance score for MTM segments: $Heat = \alpha . N_{visit}+ \beta. L_{interaction}  + \gamma . R_{recency}$                |
| **F_score**                         | Similarity measure combining cosine embedding similarity and Jaccard keyword overlap, used to assign pages to segments                   |
| **FIFO**                            | First-In-First-Out eviction policy governing the transition from STM to MTM transitions                                                  |
| **Heat-based Eviction**             | Policy that evicts the coldest (least visited/recent) MTM segments when capacity is exceeded                                             |
| **Segmented Paging**                | OS-borrowed strategy: logical memory is divided into named segments, each subdivided into fixed-size pages                               |
| **User Traits**                     | 90-dimensional dynamic attribute space covering basic needs, AI alignment, and content interest tags                                     |
| **LoCoMo**                          | Long-Conversation Memory benchmark with ~300-turn dialogues (~9K tokens), evaluating Single-hop, Multi-hop, Temporal, and Open-domain QA |
| **GVD**                             | A multi-turn dialogue dataset simulating 15 virtual users over 10 days across multiple topics                                            |
| **Locality-sensitive hashing**      | A probabilistic algorithm used to efficiently find approximate nearest neighbors in massive, high-dimensional datasets                   |
| **Ebbinghaus forgetting curve**     | Developed by psychologist Hermann Ebbinghaus in 1885, illustrates that humans forget information exponentially fast                      |

---
# 2. Paper Summary (What)

MemoryOS is a memory management framework for LLM-based agents that mirrors OS-level memory design. It organises conversational history across three hierarchical tiers (STM, MTM, LPM) and coordinates four modules (Storage, Updating, Retrieval, and Generation) to produce coherent, personalised responses across arbitrarily long interactions. 

---
# 3. What it Solves (Why)

- LLMs rely on fixed-length context windows, making them unable to preserve continuity across sessions with temporal gaps
	=> factual inconsistencies, loss of user-specific preferences, and no persistent agent persona. 

- No prior system provides a _unified operating system_ integrating storage architecture, update policy, retrieval strategy, and persona management under one coherent framework (MemGPT does not have heat-based prioritisation)

---
# 4. Methodology (How)
![[MemoryOS.pdf#page=3&rect=53,497,546,779&color=yellow|MemoryOS, p.3]]

MemoryOS is composed of four coordinated modules:
## 4.1. Memory Storage:
- Organizing and storing information by a three-tier hierarchical structure:
	- **STM** holds the last 7 dialogue pages in a FIFO queue. Each page carries a dialogue chain link (meta information) which is a summary of all of its chained pages (LLM decides chain linage).
	- **MTM** organises pages into topic segments using F_score (cosine similarity + Jacard). A page is assigned to a segment if F_score > $\theta$ (default 0.6); otherwise a new segment is created.
		$$ F_{Jacard} = \frac{|K_s \cap K_p|}{|K_s \cup K_p|}

			$$
	- **LPM** stores:
		- User Persona:
			- static User Profiles (gender, name,...)
			- User knowledge base: FIFO queue of 100 facts
			- User Traits: 90-dimensional evolving vector
		-  Agent Persona:
			- static Agent Profile: fixed setting (roles, character traits) 
			- Agent Traits: FIFO queue of 100 facts (recommended items)
## 4.2. Memory Updating
- Mange dynamic memory refreshing:
	- **_STM → MTM_:** When STM is full, the oldest page is popped and matched to an MTM segment via F_score -> Update that segment $L_{interaction}$ in MTM.
	- **_MTM → LPM_:** When a segment's Heat > $\tau$ (=5), the segment is distilled into LPM updates (User KB entries, User Traits, Agent Traits). After transfer, $L_{interaction}$ resets to 0, cooling the segment.
				$Heat = \alpha . N_{visit}+ \beta. L_{interaction}  + \gamma . R_{recency}$
				- $N_{visit}$ : Number of times the segment has been retrieved
				- $L_{interaction}$: total number of dialogue pages within the segment
				- $R_{recency} = exp(-\frac{\Delta t}{\micro})$: Duration since last retrieval time
	- **_MTM eviction_:** When MTM is at capacity, segments with the lowest Heat are deleted.
## 4.3. Memory Retrieval: 
- A three-source retrieval is fused at generation time:
	- **STM**: all pages returned verbatim (recency context).
	- **MTM**: two-stage:
		1. top-_m_ segments by F_score 
		2. top-_k_ pages within those segments by semantic similarity.
			-> update $N_{visit}$ and $R_{recency}$ of retrieved segments
	- **LPM**: 
		- top-10 User KB entries and top-10 Agent Trait entries by semantic similarity
		- full User Profile, Agent Profile and User Traits always included.
## 4.4. Response Generation:
The final prompt concatenates STM pages, retrieved MTM pages, and the full LPM snapshot, then passes it to the backbone LLM.

**Worked Example** _(User: "Do you remember what I did last weekend?")_

| Step                         | Action                                                                                       | Memory State                                           |
| ---------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| New turn arrives             | Page₁₅ = {Q:"last weekend?", R:"...", T:t₁₅} pushed to STM                                   | STM: [p₉…p₁₅]                                          |
| STM full (cap=7)             | p₉ popped; $F_{score}$(p₉, seg_"park") = 0.72 > 0.6                                          | p₉ merged into seg_"park"                              |
| Heat(seg_"park") = 6.1 > τ=5 | Segment distilled; "User visited wetland park, ran 2 laps, saw squirrels" written to User KB | LPM updated                                            |
| Query retrieval              | - STM: p₁₀–p₁₅ returned<br>- MTM: top-2 segments → top-10 pages<br>- LPM: relevant KB facts  | All 3 tiers fused                                      |
| Generation                   | Prompt = STM context + MTM pages + LPM facts + Q                                             | Response recalls park visit with fitness goal reminder |

---

## 4.5. Benchmarks

### 4.5.1. Other Baselines

| Method                        | Core Idea                                   | Limitation                                                          |
| ----------------------------- | ------------------------------------------- | ------------------------------------------------------------------- |
| **TiM** ([[Think-in-Memory]]) | Stores reasoning chains; retrieves via LSH  | Single-stage hash retrieval misses cross-topic dependencies         |
| **[[MemoryBank]]**            | Ebbinghaus forgetting curve + vector DB     | Decay alone insufficient; no structural organisation                |
| **[[MemGPT]]**                | OS-style dual-tier context with read/write  | Flat FIFO causes topic mixing at scale                              |
| **[[A-Mem]]**                 | Dynamic note graph with inter-session links | High latency and error accumulation from multi-step link generation |

### 4.5.2. Benchmarks

| Benchmark      | Description                                                                                                  | Metrics                                                                                                             |
| -------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| **[[GVD]]**    | 15 virtual users, 10-day simulation, 2+ topics/day, multi-turn                                               | Retrieval Accuracy (0 or 1), Response Correctness (0/0.5/1), Contextual Coherence (0/0.5/1), scored by DeepSeek-R1) |
| **[[LoCoMo]]** | Ultra-long dialogues (~300 turns, ~9K tokens); 4 QA categories: Single-hop, Multi-hop, Temporal, Open-domain | F1, BLEU-1                                                                                                          |

### 4.5.3. Notable Results

- On **LoCoMo (GPT-4o-mini)**: MemoryOS achieves **+49.11% avg F1** and **+46.18% avg BLEU-1** over the best baseline; most pronounced on Temporal (+118.8% F1), the hardest category for all baselines.
- On **GVD (GPT-4o-mini)**: +3.2% Accuracy, +5.4% Correctness, +1.0% Coherence over A-Mem.
- **Efficiency**: 4.9 avg LLM calls vs. A-Mem's 13; 3,874 tokens consumed vs. MemGPT's 16,977
- **Ablation**: Removing MTM causes the largest drop; removing LPM the second largest; removing dialogue chain has the least impact.

---

## 4.6. Strengths

- **Unified OS abstraction**: First to integrate storage, update policy, retrieval, and persona management into a single coherent framework rather than optimising one dimension.
- **Heat-based eviction**: Combines visit frequency, engagement depth, and recency decay into one prioritisation signal, giving principled control over what persists.
- **Efficiency-accuracy trade-off**: Achieves state-of-the-art accuracy with one of the lowest LLM call counts (4.9), making it practically deployable.
- **Persona persistence**: The 90-dimensional User Traits + User KB enables proactive, goal-aware responses (e.g., reminding users of fitness targets when ordering food).
- **Modular design**: Ablation-verified independent contribution of each module (MTM, LPM, Chain) facilitates targeted extension or replacement.

---

## 4.7. Gaps

1. **θ and τ is manually tuned**: F_score threshold (θ=0.6) and Heat threshold (τ=5) are manually tuned; no adaptive or learned calibration is proposed, which may hurt generalisation across domains.
2. **Cold-start problem with LPM**: LPM starts empty; early interactions are purely STM/MTM-driven, and the system requires extended dialogue before persona benefits materialise.
3. **Limited evaluation domain**: Experiments are limited to two English datasets; performance on code-switching, multi-lingual, or domain-specific (medical, legal) dialogues is unknown.
4. **Backbone dependency**: The system relies on LLM calls for chain summarisation, segment creation, trait extraction, and response generation 

---

## 4.8. Highlights

> [!PDF|255, 208, 0] [[MemoryOS.pdf#page=1&annotation=513R|MemoryOS, p.1]]
> > memory mechanisms in default LLMs can be broadly categorized into three methodological types

> [!PDF|yellow] [[MemoryOS.pdf#page=4&selection=30,20,40,1&color=yellow|MemoryOS, p.4]]
> > rst, evaluating a new page’s contextual relevance to prior pages to determine chain linkage or resetting to the current page if semantically discontinuous; second, summarizing all chain pages into metachain i .

> [!PDF|yellow] [[MemoryOS.pdf#page=4&selection=319,0,366,45&color=yellow|MemoryOS, p.4]]
> > Nvisit is the number of times the segment has been retrieved, Linteraction denotes the total number of dialogue pages within the segment, and Rrecency is the time decay coefficient represents the duration since the last retrieval time of the current segment, defined as: Rrecency = exp  − ∆t μ  , where ∆t is the time elapsed since the last access, measured in seconds, and μ is a configurable time constant (i.e., 1e+7).

> [!PDF|yellow] [[MemoryOS.pdf#page=5&selection=26,10,29,27&color=yellow|MemoryOS, p.5]]
> >  factual information relevant to the user and agent assistant is extracted and recorded into the User KB and Agent Traits, respectively.

> [!PDF|yellow] [[MemoryOS.pdf#page=5&selection=34,0,38,36&color=yellow|MemoryOS, p.5]]
> > Linteraction in Eq. 4 is reset to zero, causing the heat score of the segment to decline

> [!PDF|yellow] [[MemoryOS.pdf#page=7&selection=480,21,495,5&color=yellow|MemoryOS, p.7]]
> > our method outperforms the Top-2 baselines (i.e., MemGPT and A-Mem) in both aspects, requiring significantly fewer LLM calls than A-Mem* (4.9 vs.13) and much lower token consumption than MemGPT (3,874 vs. 16,97
> 
> 