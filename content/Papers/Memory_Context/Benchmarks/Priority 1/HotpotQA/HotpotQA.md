---
Tags:
Date: "2018"
Authors: Zhilin Yang, Peng Qi, Saizheng Zhang, Yoshua Bengio, William W. Cohen, Ruslan Salakhutdinov, Christopher D. Manning
Venue: EMNLP
Paper: "HotpotQA: A Dataset for Diverse, Explainable Multi-hop Question Answering"
---
# 1. Terminology

| Term                            | Definition                                                                                                                                                                                                                                                                                   |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Multi-hop Reasoning**         | Reasoning that requires synthesizing information from two or more separate documents or passages to arrive at an answer                                                                                                                                                                      |
| **Bridge Entity**               | An intermediate entity that connects two documents; identified in the first hop and used to unlock the second hop <br><br>e.g.," When was the singer of Radiohead born? " --> resolving "the singer of Radiohead" → "Thom Yorke" before finding his birthday (Thom Yorke is a bridge entity) |
| **Supporting Facts**            | Sentence-level annotations indicating exactly which sentences are necessary and sufficient to answer a question                                                                                                                                                                              |
| **Distractor Setting**          | Benchmark setting where 2 gold paragraphs are mixed with 8 TF-IDF-retrieved distractor paragraphs (10 total), forcing the model to identify relevant context                                                                                                                                 |
| **Full Wiki Setting**           | Benchmark setting where the model must retrieve and reason over the first paragraphs of all ~5M English Wikipedia articles with no gold paragraph provided                                                                                                                                   |
| **Comparison Question**         | A novel question type requiring the model to compare two entities on a shared property (e.g., age, number of members, nationality), sometimes requiring arithmetic                                                                                                                           |
| **Yes/No Question**             | A subtype of comparison question where the answer is a binary judgment (e.g., "Are Iron Maiden and AC/DC from the same country?") requiring reasoning over both paragraphs                                                                                                                   |
| **CQW (Central Question Word)** | The primary question word used to heuristically classify question type (WH-words, copulas like "is/are", auxiliary verbs like "does/did")                                                                                                                                                    |
| **TF-IDF Retrieval**            | Bigram term-frequency-inverse-document-frequency used to retrieve distractor and candidate paragraphs in the two benchmark settings                                                                                                                                                          |
| **Joint Metric**                | A combined evaluation metric that multiplies precision/recall of answer span prediction and supporting fact prediction, penalizing systems weak on either task                                                                                                                               |
| **Strong Supervision**          | Using ground-truth supporting fact annotations as an auxiliary training signal via multi-task learning, in addition to the answer supervision                                                                                                                                                |
| **Train-Easy**                  | Subset of ~18k examples where crowd worker responses suggest single-hop reasoning suffices; used for training but not for dev/test                                                                                                                                                           |
| **Train-Medium**                | ~56k examples correctly answered by the baseline model with high confidence; contains more Type II questions than the hard splits                                                                                                                                                            |
| **Train-Hard / Dev / Test**     | The challenging multi-hop examples that the baseline model failed on; used for dev and test evaluation                                                                                                                                                                                       |
| **Wikipedia Hyperlink Graph G** | A directed graph where each edge (a, b) indicates a hyperlink from the first paragraph of article a to article b; the backbone of the data collection pipeline                                                                                                                               |

---

# 2. Metadata

|Property|Value|
|---|---|
|**Total examples**|112,779 QA pairs|
|**Source corpus**|English Wikipedia dump (Oct 1, 2017), ~5M articles|
|**Collection platform**|Amazon Mechanical Turk via ParlAI interface|
|**Answer format**|Extractive span (majority) + Yes/No (minority)|
|**Annotation**|Question, answer span, sentence-level supporting facts|

### 2.1.1. Data Splits

|Split|Usage|Examples|
|---|---|---|
|train-easy|Training (mostly single-hop)|18,089|
|train-medium|Training (multi-hop, model-solvable)|56,814|
|train-hard|Training (hard multi-hop)|15,661|
|dev|Development (hard multi-hop)|7,405|
|test-distractor|Test — distractor setting|7,405|
|test-fullwiki|Test — full wiki setting|7,405|
|**Total**||**112,779**|

> **Note:** Separate test sets for the two benchmark settings prevent gold paragraph leakage from the distractor setting into the full wiki evaluation.

---

# 3. What it Measures

## 3.1. Task Definition

HotpotQA tests a QA system's ability to perform **multi-hop reasoning**:  locating and synthesizing information across multiple Wikipedia paragraphs to produce both a correct answer and a faithful explanation.

Given a question $q$ and a set of context paragraphs $P = {p_1, ..., p_n}$, the model must:

1. **Answer** $q$ by extracting a text span (or predicting yes/no) from $P$
2. **Explain** the answer by identifying the subset of sentences $S^* \subseteq P$ that constitute the supporting facts

The task is evaluated jointly: a correct answer with a wrong explanation is penalized.

**Question categories by reasoning structure:**

| Type                                 | % (dev/test) | Description                                                               |
| ------------------------------------ | ------------ | ------------------------------------------------------------------------- |
| Type I — Chain reasoning             | 42%          | Identify a bridge entity in hop 1, then answer a second question about it |
| Comparison                           | 27%          | Compare two entities on a shared property; may require arithmetic         |
| Type II — Intersection               | 15%          | Locate an entity satisfying multiple properties simultaneously            |
| Type III — Bridge property inference | 6%           | Infer a property of the main entity via a collocated bridge entity        |
| Other (3+ supporting facts)          | 2%           | Requires more than two supporting facts                                   |
| Single-hop / Unanswerable            | ~8%          | Not the primary focus                                                     |

## 3.2. Example Records

**Example 1 — Type I (Bridge Entity / Chain Reasoning)**

> **Context A** _(Return to Olympus)_: "Return to Olympus is the only album by the alternative rock band **Malfunkshun**. It was released after the band had broken up and after lead singer **Andrew Wood** (later of Mother Love Bone) had died of a drug overdose in 1990."
> 
> **Context B** _(Mother Love Bone)_: "Mother Love Bone was an American rock band that formed in Seattle, Washington in 1987. Frontman **Andrew Wood**'s personality and compositions helped to catapult the group to the top of the Seattle music scene. **Wood died only days before the scheduled release of the band's debut album, 'Apple'**, thus ending the group's hopes of success."
> 
> **Q:** What was the former band of the member of Mother Love Bone who died just before the release of "Apple"?
> 
> **A:** Malfunkshun
> 
> **Supporting facts:** Sentences 1, 2 (Context A) + Sentences 4, 6, 7 (Context B)

_Bridge entity: Andrew Wood. Hop 1 identifies who died before "Apple" release. Hop 2 identifies his former band._

---

**Example 2 — Comparison (with arithmetic)**

> **Context A** _(LostAlone)_: "LostAlone were a British rock band... consisted of **Steven Battelle, Alan Williamson, and Mark Gibson**..."
> 
> **Context B** _(Guster)_: "Guster is an American alternative rock band... Founding members **Adam Gardner, Ryan Miller, and Brian Rosenworcel** began..."
> 
> **Q:** Did LostAlone and Guster have the same number of members?
> 
> **A:** Yes
> 
> _Both have 3 members  requires counting across two documents._

---

**Example 3 — Type II (Intersection / Multiple Properties)**

> **Context A** _(Pittsburgh Pirates members)_: "Several current and former members of the Pittsburgh Pirates... John Milner, Dave Parker, and Rod Scurry..."
> 
> **Context B** _(David Gene Parker)_: "David Gene Parker, nicknamed **'The Cobra'**, is an American former player in Major League Baseball..."
> 
> **Q:** Which former member of the Pittsburgh Pirates was nicknamed "The Cobra"?
> 
> **A:** Dave Parker
> 
> _Must satisfy both: (1) former Pirates member AND (2) nicknamed "The Cobra"._

## 3.3. Metrics

HotpotQA defines three evaluation dimensions, each with EM and F1:

- **Answer Span Metrics**: token-level F1, exact string match following SQuAD convention.

- **Supporting Fact Metrics**:
	- **Sup Fact EM**: 1 only if the predicted set exactly equals the gold set
	- **Sup Fact F1**: token-level overlap between predicted and gold supporting sentences

- **Joint Metrics**:

		$$P^{(\text{joint})} = P^{(\text{ans})} \cdot P^{(\text{sup})}, \quad R^{(\text{joint})} = R^{(\text{ans})} \cdot R^{(\text{sup})}$$
		$$\text{Joint F1} = \frac{2 P^{(\text{joint})} R^{(\text{joint})}}{P^{(\text{joint})} + R^{(\text{joint})}}$$

	- **Joint EM = 1** only when both answer and supporting facts achieve exact match simultaneously.

---
# 4. Uniqueness (Why)

## 4.1. Other benchmarks:
- [[SQuAD]]: single-hop QA
- [[TriviaQA]] : single-hop QA
- [[SearchQA]]: single-hop QA
- [[QAngaroo]]: derive questions from KB schemas (Freebase, Wikidata)
- [[ComplexWebQuestions]]: derive questions from KB schemas (Freebase, Wikidata)

## 4.2. Uniqueness

**1. Beyond single-hop reading comprehension:**  HotpotQA explicitly requires fusing evidence from _at least two_ documents, with crowd workers instructed to compose questions that genuinely demand both.

**2. Beyond knowledge-base constraints:** HotpotQA is grounded purely in free text, enabling diverse natural-language questions about any Wikipedia-expressible fact — not just KB-representable ones.

**3. Explainability by design:** HotpotQA is the first large-scale QA dataset to collect and release **sentence-level supporting facts** alongside answers.

**4. Novel question type.** Text-based **comparison questions** requiring arithmetic (counting, ordering, numerical comparison) had no dedicated dataset prior to HotpotQA. 

---
# 5. Generation Method (How)

The collection pipeline is carefully engineered to ensure multi-hop questions emerge naturally:

1. **Build a Wikipedia Hyperlink Graph.** Extract all hyperlinks from the _first paragraph_ of every Wikipedia article. Construct directed graph $G$ where edge $(a, b)$ means article $a$'s first paragraph links to article $b$.

2. **Curate Bridge Entities:** 591 categories are manually selected from WikiProject's popular pages lists. Bridge entity candidates $B$ are drawn only from these curated categories.
	For _comparison questions_: 42 lists of semantically similar entities (e.g., "Highest Mountains on Earth") are manually curated from Wikipedia's "list of lists." Two paragraphs are randomly sampled from the same list.

3. **Generate Candidate Paragraph Pairs:** Sample edges $(a, b)$ from $G$ with $b \in B$ → present paragraphs $a$ and $b$ to crowd workers.  

4. **Crowd Collection via ParlAI.** Workers on Amazon Mechanical Turk are shown the two paragraphs and asked to write a question that requires both. Workers also annotate the supporting facts. A **bonus structure** rewards top contributors every 200 examples and by productivity (examples/hour).

5. **Quality Filtering and Split Assignment.** Questions are split into easy/medium/hard via model-in-the-loop evaluation: a baseline QA model identifies questions it can answer confidently (→ medium) or cannot (→ hard). A manual sample from top Turkers identifies predominantly single-hop examples (→ easy). 

6. **Create hard benchmarks:** Hard examples are split into *train-hard, dev, test-distractor, and test-fullwiki*.
	- **test-distractor:** Use bigram TF-IDF to retrieve 8 paragraphs from Wiki as distractors and shuffle with 2 goldens.
	- **fullwiki**: Gather all first paragraphs of Wiki articles.

---

# 6. Contributions

1. **First large-scale, free-text multi-hop QA dataset** (113k examples) not constrained to any KB schema, enabling natural and diverse multi-hop questions across all of Wikipedia.

2. **Sentence-level supporting fact annotations** released as part of the dataset. The first QA benchmark to provide strong, sentence-level explainability supervision at scale.

3. **Joint evaluation framework** (Joint EM / Joint F1) that simultaneously penalizes wrong answers _and_ wrong explanations, raising the bar beyond answer-only metrics.

4. **Novel comparison question type** requiring arithmetic reasoning (counting, age comparison, numerical ordering) over two entities, a capability untested by prior benchmarks.

5. **Two-setting evaluation protocol** (distractor vs. full wiki) that separately assesses reading comprehension ability and end-to-end retrieval+reasoning ability, providing finer-grained diagnostic power.

---

# 7. Gaps

1. **Simple answer format.** The task is extractive span or yes/no. Abstractive, generative, or multi-sentence answers are outside the benchmark's scope.
	=> Enhance with new complex question sets

2. **Wikipedia-only domain:** Generalization to other domains (scientific papers, legal text, dialogues) is not tested.
	=> Enhance with new datasources

3. **Annotation subjectivity in supporting facts.** Human upper bound for supporting fact EM is only 87.4% (vs. ~97% for answers), indicating inherent annotator disagreement about which sentences are "necessary." This injects noise into explainability evaluation.

4. **No unanswerable question track.** Only ~2% of examples are unanswerable, and these are not a formal evaluation category. Robustness to unanswerable or adversarial questions (a key SQuAD 2.0 contribution) is out of scope.
	=> Explore the unanswerable records to have new insights into LLM's reasoning capabilities 

# 8. Highlights
> [!PDF|255, 208, 0] [[HotpotQA A Dataset for Diverse Explainable Multi-hop Question Answering.pdf#page=2&annotation=453R|HotpotQA A Dataset for Diverse Explainable Multi-hop Question Answering, p.2]]
> > simply giving an arbitrary set of paragraphs to crowd workers is counterproductive, because for most paragraph sets, it is difficult to ask a meaningful multi-hop question.


> [!PDF|255, 208, 0] [[HotpotQA A Dataset for Diverse Explainable Multi-hop Question Answering.pdf#page=3&annotation=474R|HotpotQA A Dataset for Diverse Explainable Multi-hop Question Answering, p.3]]
> > To the best of our knowledge, text-based comparison questions are a novel type of questions that have not been considered 

> [!PDF|yellow] [[HotpotQA A Dataset for Diverse Explainable Multi-hop Question Answering.pdf#page=8&selection=254,3,257,20&color=yellow|HotpotQA A Dataset for Diverse Explainable Multi-hop Question Answering, p.8]]
> > our proposed method of incorporating supporting facts supervision is most likely suboptimal, and we leave the challenge of better modeling to future work. 
> 
> 