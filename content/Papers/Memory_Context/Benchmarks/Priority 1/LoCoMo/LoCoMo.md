---
Tags:
  - Benchmark
Date: "2024"
Authors: A Maharana, DH Lee, S Tulyakov, M Bansal, F Barbieri, Y Fang
Venue: ACL
Paper: Evaluating Very Long-Term Conversational Memory of LLM Agents
---
# 1. Terminology

| Term                                     | Definition                                                                                                                                                     |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **LoCoMo**                               | Long-term Conversational Memory: the dataset and benchmark introduced in this paper, comprising very long multi-session, multi-modal dialogues                 |
| **Session**                              | A single conversation block between two agents occurring on a specific date; multiple sessions form one full conversation                                      |
| **Temporal Event Graph (G)**             | A directed graph per agent where nodes are life events $e_i$​ with timestamps $t_i$​, and edges $l=(e_i, e_j)$ encode causal relationships between events      |
| **Persona (p)**                          | A rich natural-language profile for each virtual agent, covering name, age, habits, relationships, and objectives; expanded from short MSC seed statements     |
| **Reflect & Respond**                    | The agent mechanism that conditions response generation on both short-term (session summary $w_k$​) and long-term (observation store $\mathcal{H}_l$ ​) memory |
| **Observation ($o_{k_j}$​​)**            | An atomic, assertive factual statement about a speaker extracted from a single dialogue turn and stored in long-term memory                                    |
| **Short-term Memory ($\mathcal{H}_s$​)** | A rolling session-level summary $w_k$​ that carries forward the main ideas of past sessions                                                                    |
| **Long-term Memory ($\mathcal{H}_l$ ​)** | A database of per-turn observations retrieved at inference time via a dense retriever                                                                          |
| **FactScore**                            | A factuality metric that decomposes text into atomic facts and measures precision/recall against a reference set of atomic facts                               |
| **MMRelevance**                          | A multimodal relevance metric measuring alignment between predicted and ground-truth image-text dialogue pairs                                                 |
| **Adversarial QA**                       | Questions designed to trick the model into answering; the correct response is "unanswerable"                                                                   |
| **DRAGON**                               | Dense Retrieval Augmented with Gold and Negative samples: the retriever model used in RAG experiments                                                          |

---
# 2. Metadata

| Property                                        | Value                                                                                                        |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Total conversations**                         | 10                                                                                                           |
| **Avg. sessions per conversation**              | 27.2                                                                                                         |
| **Avg. turns per session**                      | 21.6                                                                                                         |
| **Avg. turns per conversation**                 | ~588                                                                                                         |
| Avg. tokens per conversation                    | 16,618                                                                                                       |
| Avg. tokens per turn (dialogue)                 | 29.8                                                                                                         |
| Avg. tokens per observation                     | 19.2                                                                                                         |
| Avg. tokens per session summary                 | 132.4                                                                                                        |
| **Avg. events per conversation (ground truth)** | 35.8                                                                                                         |
| **Avg. images per conversation**                | 91.2                                                                                                         |
| **Total QA questions**                          | 1,986                                                                                                        |
| **QA split**                                    | - Single-hop 42.3% <br>- Multi-hop 14.2% <br>- Temporal 16.1% <br>- Open-domain 4.8% <br>- Adversarial 22.4% |
| **Avg. tokens per event summary**               | 1,042.7                                                                                                      |
| **Modality**                                    | Text + Images (web-sourced)                                                                                  |
| **Language**                                    | English only                                                                                                 |
| **Collection method**                           | LLM-generated (GPT-3.5-turbo / text-davinci-003) + crowdsourced human verification                           |
| **Time span per conversation**                  | A few months (event graphs span 6–12 months)                                                                 |
| **Input format**                                | Multi-turn dialogue text interleaved with image captions (or raw images for multimodal task)                 |
| **Output format**                               | - Short answer spans (QA) <br>- Event list (summarization) <br>- Text + image (multimodal generation)        |

---

## 3. What it measures (What)

### 3.1. Task definition

LoCoMo (Long-term Conversational Memory) evaluates three complementary capabilities of long-term dialogue agents:

- **Question Answering (QA).** Given the full conversation history, the model must answer questions drawn from five reasoning categories: 
	1.  _single-hop_: one fact from one session
	2.  _multi-hop_: synthesis across sessions
	3. _temporal_: reasoning about time-ordered events\
	4. _open-domain knowledge_: integrating dialogue facts with commonsense or world knowledge
	5. _adversarial_: correctly identifying unanswerable questions designed to induce hallucination.

- **Event Summarization.** Given the full conversation history, the model must summarize the significant life events of a designated speaker within a specified timeframe. The ground truth is the temporal event graph $\mathcal{G}$, decomposed into atomic facts for evaluation.

- **Multimodal Dialogue Generation.** Given preceding dialogue context (text + images), the model must generate the next response as an interleaved text-and-image turn. This tests persona consistency and narrative coherence across weeks or months of conversational history.

### 3.2. Example Records of the dataset

![[LoCoMo.pdf#page=2&rect=78,418,514,647&color=yellow|LoCoMo, p.13852]]

### 3.3. Metrics

| Task                  | Metric                                  | Details                                                                          |
| --------------------- | --------------------------------------- | -------------------------------------------------------------------------------- |
| QA                    | **F1 (token-level partial match)**      | Normalized exact-match F1 between predicted and ground-truth answer spans        |
| QA (RAG only)         | **Recall@k**                            | Whether the correct source turn appears in the top-k retrieved passages          |
| Event Summarization   | **ROUGE-1/2/L**                         | Lexical overlap between predicted and reference event summaries                  |
| Event Summarization   | **FactScore (Precision / Recall / F1)** | Atomic-fact decomposition measuring factual accuracy rather than surface overlap |
| Multimodal Generation | **BLEU-1/2**                            | N-gram precision on generated text                                               |
| Multimodal Generation | **ROUGE-L**                             | Longest common subsequence overlap on generated text                             |
| Multimodal Generation | **MMRelevance**                         | Semantic alignment between predicted and ground-truth image-dialogue pairs       |

---

## 4. Uniqueness (Why)

### 4.1. Other benchmarks

| Dataset                     | Avg. turns | Avg. sessions | Avg. tokens  | Multimodal | Collection                   |
| --------------------------- | ---------- | ------------- | ------------ | ---------- | ---------------------------- |
| [[MPChat]]                  | 2.8        | 1             | 53.3         | ✓          | Reddit                       |
| [[MMDialog]]                | 4.6        | 1             | 72.5         | ✓          | Social media                 |
| [[Multi-Session Chat]]      | 53.3       | 4             | 1,225.9      | ✗          | Crowdsourcing                |
| [[Conversation Chronicles]] | 58.5       | 5             | 1,054.7      | ✗          | LLM-generated                |
| **LoCoMo**                  | **588.2**  | **27.2**      | **16,618.1** | **✓**      | **LLM-gen. + Crowdsourcing** |


### 4.2. Uniqueness

LoCoMo distinguishes itself along four axes:
- **Scale.** At ~600 turns and ~16K tokens per conversation across up to 32 sessions, it is 16× longer in tokens and 5× richer in sessions than the closest prior work (MSC).
- **Causal-temporal grounding.** Dialogues are not free-form but are explicitly anchored to temporal event graphs $\mathcal{G}$ with causal edges, enabling evaluation of whether models track *why* and *when* events happen.
- **Multimodal long-term dialogue.** It is the first benchmark to combine image-sharing and image-reaction behaviors in very long-term conversations, enabling evaluation of visual long-term consistency at scale.
- **Hybrid quality control.** Unlike purely LLM-generated benchmarks, every conversation is verified and edited by human annotators (~15% of turns edited, ~19% of images removed or replaced)

---

## 5. Generation Method (How)

The dataset is constructed through a four-stage human-machine pipeline:

1. **Stage 1 - Persona creation.** Seed personas ($p_c$​, 4–5 sentences) are sampled from the MSC dataset and expanded by GPT-3.5-turbo into rich persona statements $p$ covering name, age, habits, relationships, and objectives.

2. **Stage 2 - Temporal Event Graph construction.** 
	- For each agent, `text-davinci-003` generates a directed event graph $\mathcal{G}$ of up to 25 causally linked events spread over 6-12 months.
	- Nodes are life events $e_i$​ with timestamps $t_i$​, and edges $l=(e_i, e_j)$ encode causal relationships between events
	- Events are generated in batches of $k=3$ iteratively, where each new batch is causally conditioned on previous events, ensuring a realistic life trajectory.

4. **Stage 3 - Virtual agent dialogue generation.** Two LLM agents ($\mathcal{L}_1$ ​, $\mathcal{L}_2$​, both `GPT-3.5-turbo`) converse using a generative agent architecture with two mechanisms:
	- ***Reflect & Respond***: 
		- At each turn, the agent retrieves relevant long-term observations $o$ and conditions on the latest session summary $w_k$ (stored in short term memory), current chat history $h_{k+1}$ ​, persona $p$, and the subset of events $\{e \in \mathcal{G} \mid t^s_k < t^e_i < t^s_{k+1}\}$ that occurred  between the last and current session. 
		- After each turn, the turn is transformed into observations stored in long-term memory. 
		- After each session, a summary is extracted to be stored in short term memory.
	- **_Image sharing & reaction_**: 
		- Sharing: the agent generates an image caption for the intended image → extracts keywords → crawls the web for an image → shares it
		- Reaction: upon receiving an image BLIP-2 generates a caption and the agent reacts accordingly.
	
5. **Stage 4 - Human verification and editing.** Annotators are tasked with:
	- fix long-range inconsistencies (~15% of turns edited)
	- remove or replace irrelevant images (~19%)
	- verify alignment between dialogue content and the event graphs.

---

## 6. Gaps

1. **Tiny dataset size (n=10).** Only 10 conversations exist in LoCoMo, making statistical significance of benchmark results questionable and limiting sub-group analysis (e.g., across persona types or topic domains).
2. **No real-world conversation.** All dialogues are agent-generated and may not capture the full range of pragmatic phenomena found in human–human long-term conversations (topic drift, social repair, code-switching, emotional escalation).
3. **Weak visual grounding.** Web-crawled images lack personal visual continuity (consistent faces, home environments, pets), which means images can largely be substituted by their captions without much performance loss 
4. **Adversarial questions are binary (answerable/not).** The adversarial category only probes whether models refuse to hallucinate, missing richer adversarial dimensions such as temporal contradiction, persona inconsistency injection, or cross-speaker attribution attacks.
5. **No persona drift or contradiction injection.** The benchmark lacks adversarial cases where a speaker's stated persona evolves or contradicts earlier turns, which would stress-test models' ability to maintain coherent, up-to-date user models over time.

---

# 7. Highlights

> [!PDF|255, 208, 0] [[LoCoMo.pdf#page=1&annotation=2538R|LoCoMo, p.13851]]
> > Existing works on long-term open-domain dialogues focus on evaluating model responses within contexts spanning no more than five chat sessions

> [!PDF|255, 208, 0] [[LoCoMo.pdf#page=1&annotation=2541R|LoCoMo, p.13851]]
> > equip each agent with the capability of sharing and reacting to images.

> [!PDF|255, 208, 0] [[LoCoMo.pdf#page=1&annotation=2544R|LoCoMo, p.13851]]
> > verified and edited by human annotators for long-range consistency and grounding to the event graphs

> [!PDF|255, 208, 0] [[LoCoMo.pdf#page=3&annotation=2559R|LoCoMo, p.13853]]
> > five distinct reasoning types to evaluate memory from multiple perspectives: single-hop, multi-hop, temporal, commonsense or world knowledge, and adversarial. 

> [!PDF|255, 208, 0] [[LoCoMo.pdf#page=3&annotation=2562R|LoCoMo, p.13853]]
> > In this task, the event graphs linked to each LLM speaker serve as the correct answers, and models are tasked with extracting this information from the conversation history. 
> 
> 

> [!PDF|255, 208, 0] [[LoCoMo.pdf#page=3&annotation=2565R|LoCoMo, p.13853]]
> > Conversational agents need to utilize relevant context recalled from past conversations to generate responses that are consistent with the ongoing narrative. We assess this ability via the multi-modal dialog generation task


> [!PDF|255, 208, 0] [[LoCoMo.pdf#page=4&annotation=2568R|LoCoMo, p.13854]]
> > The generated statements typically include details about one or more of the following elements (Gao et al., 2023a): objectives, past experiences, daily habits, and interpersonal relationships, as well as name, age, and gender of the individual


