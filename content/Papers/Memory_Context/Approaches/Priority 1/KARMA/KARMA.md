---
Tags:
Date: "2026"
Authors: Yuxing Lu, Wei Wu, Xukai Zhao, Rui Peng, Jinzhuo Wang (Peing, Tsinghua)
Venue: NeurIPS
Paper: "KARMA: Leveraging Multi-Agent LLMs for Automated Knowledge Graph Enrichment"
Memory type:
  - Token-level
Agent env: Multi-agent
Record format: Text
Memory architecture:
  - graph-a
Tackle Module: KG enrichment
Need offline initialization: true
Fine-tuning?: false
Other tags:
---
# 1. Terminology

| Term                               | Definition                                                                                                                                                                                                                                                       |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Knowledge Graph (KG)**           | A structured data representation where _entities_ (e.g., genes, drugs, diseases) are nodes and _relationships_ (e.g., "treats", "causes") are directed edges. <br><br>E.g., `Aspirin --treats--> Headache`. Used by Wikidata, DBpedia, and biomedical databases. |
| **KG Enrichment**                  | The process of expanding an existing KG by extracting new entities and relations from external sources (e.g., scientific papers) and integrating them while preserving consistency.                                                                              |
| **Triplet**                        | The atomic unit of a KG: `(head entity, relation, tail entity)`. <br><br>E.g., `(EGFR, causes, lung adenocarcinoma)`. All extracted knowledge is expressed as triplets.                                                                                          |
| **Entity Normalization**           | Mapping variant surface forms of an entity to a canonical identifier. <br><br>E.g., "acetylsalicylic acid" and "aspirin" both map to `MESH:D001241` in a medical ontology.                                                                                       |
| **Schema Alignment**               | Mapping newly discovered entities or relations to the existing type taxonomy of a KG (e.g., deciding that "CRISPR-Cas9" belongs to type `Gene-Editing Tool`). Prevents ontological fragmentation.                                                                |
| **Conflict Resolution**            | Detecting and solving contradictions between a new triplet and existing KG knowledge. <br><br>E.g., if the KG says `(DrugX, treats, DiseaseY)` and a new paper says `(DrugX, causes, DiseaseY)`, a conflict agent must decide which to keep.                     |
| **Named Entity Recognition (NER)** | The NLP task of identifying spans of text that refer to entities of predefined types (Drug, Gene, Disease, etc.). Used here as the first extraction step.                                                                                                        |
| **Multi-Agent System (MAS)**       | An architecture where multiple AI agents, each specialised for a subtask, collaborate to solve a larger task. Cross-agent verification improves reliability over a single monolithic model.                                                                      |
| **MoE (Mixture of Experts)**       | A neural architecture (used by DeepSeek-v3) where only a subset of model parameters ("experts") is activated per token, enabling large parameter counts with manageable compute.                                                                                 |
| **RLHF**                           | Reinforcement Learning from Human Feedback: a fine-tuning approach (used in GPT-4o) that aligns model outputs with human preferences.                                                                                                                            |
| **Coverage Gain (ΔCov)**           | Number of new entities added to the KG that were not present before. Measures the breadth of enrichment.                                                                                                                                                         |
| **Connectivity Gain (ΔCon)**       | Net increase in node degree across existing KG entities after enrichment. Measures how much denser the graph becomes.                                                                                                                                            |
| **Conflict Ratio (RCR)**           | Fraction of candidate edges removed by the Conflict Resolution Agent.                                                                                                                                                                                            |
| **LLM-based Correctness (RLC)**    | Fraction of new triplets judged "likely correct" by a held-out LLM evaluator. Acts as a proxy for precision.                                                                                                                                                     |
| **QA Coherence (CQA)**             | Fraction of domain-specific questions that can be answered correctly by traversing the enriched KG. Measures practical downstream utility.                                                                                                                       |
| **PubMed**                         | The primary biomedical literature database (NIH), containing over 35 million citations. Used as the document corpus in this paper.                                                                                                                               |
| **Ontology**                       | A formal vocabulary defining types and relationships within a domain (e.g., UMLS, MeSH, SNOMED CT for biomedicine). Used to normalise extracted entities.                                                                                                        |

---

## 1.1. Paper Summary (What)

KARMA (**K**nowledge-**A**ugmentation via **R**easoning with **M**ulti-**A**gents) is a hierarchical, modular multi-agent LLM framework for automating the enrichment of knowledge graphs from unstructured scientific text. The system decomposes the KG enrichment pipeline into nine specialised agents (an LLM with a targeted prompt) that cooperate in three phases: **Parsing**, **Extraction**, and **Integration**.

The paper evaluates KARMA on 1,200 PubMed biomedical articles spanning three omics domains: **Genomics** (720 papers), **Proteomics** (360 papers), and **Metabolomics** (120 papers), using three LLM backbones: GLM-4, GPT-4o, and DeepSeek-v3.

**Key headline results:**
- Up to **38,230 new entities** discovered (Genomics, DeepSeek-v3)
- **83.1% LLM-verified correctness** (Genomics, DeepSeek-v3)
- **18.6% reduction in conflict edges** compared to a single-agent baseline
- Multi-agent setup outperforms single-agent on 17/24 metrics across all domains

---

# 2. What It Solves (Why)

## 2.1. The Core Problem

Scientific knowledge is growing faster than it can be manually curated into structured KGs. Over **7 million articles** are published annually, yet biomedical KGs like Wikidata and DBpedia update slowly via human expert curation, a process that does not scale.

## 2.2. Why Previous Approaches Fall Short

|Approach|Limitation|
|---|---|
|Manual curation|Reliable but unsustainable at scale|
|Rule-based NLP (e.g., WordNet, ConceptNet)|High precision, low recall; poor generalisation to new domains|
|Supervised neural NLP (e.g., BioBERT)|Requires 3–5k labelled examples per relation type; fails on novel schemas|
|Single monolithic LLM|Hallucinations, schema inconsistency across documents, quadratic compute costs on long texts|
|Prior multi-agent KG systems|Predefined pipelines; lack cross-agent verification; poor domain adaptation|
## 2.3. What KARMA Specifically Addresses

1. **Hallucination** — multi-layer cross-agent verification (Conflict Resolution + Evaluator) catches fabrications before integration.
2. **Schema inconsistency** — a dedicated Schema Alignment Agent maps every new entity/relation to the KG taxonomy.
3. **Scalability** — summarisation condenses inputs before extraction, reducing token costs; the pipeline maintains linear time complexity with respect to document length.
4. **Domain adaptability** — domain-adaptive prompting strategies allow the same framework to operate across genomics, proteomics, and metabolomics without task-specific retraining.

---
# 3. Methodology (How)

**Running example**: A new PubMed paper states —

> _"MTX-531 was shown to treat HNSCC (head and neck squamous cell carcinoma) by targeting PIK3CA mutations, with efficacy validated in PDX models."_


![[KARMA Leveraging Multi-Agent LLMs for Automated Knowledge Graph Enrichment.pdf#page=3&rect=82,439,546,728&color=yellow|KARMA Leveraging Multi-Agent LLMs for Automated Knowledge Graph Enrichment, p.3]]

## 3.1. Step 1: Ingestion Agent (IA)

Retrieves the PDF, handles OCR errors, and outputs a JSON with metadata (title, authors, DOI) and the cleaned full text.

```
Output: { "metadata": { "title": "...", "pmid": "..." },
          "content": "MTX-531 was shown to treat HNSCC ..." }
```

---
## 3.2. Step 2: Reader Agent (RA)

Segments the document (Abstract, Introduction, Results, etc.) and assigns each segment a **relevance score** R(sⱼ) via:

$$
R(s_j) = LLM_{reader}(s_j, G)
$$

Segments scoring below threshold $\delta$ are discarded (e.g., acknowledgements section get R ≈ 0.1 and are dropped). 

The Results section about MTX-531 scores R = 0.92 and is passed on.

---
## 3.3. Step 3: Summarizer Agent (SA)

Condenses high-relevance segments to ≤100 words while preserving domain terms and numeric data.

```
Input:  "MTX-531 was shown to treat HNSCC by targeting PIK3CA mutations..."
Output: "MTX-531 treats HNSCC; mechanism involves PIK3CA mutation targeting;
         efficacy shown in PDX models."
```

This reduces token load for downstream agents without losing biomedical precision.

---
## 3.4. Step 4: Entity Extraction Agent (EEA)

Runs LLM-based NER on the summary and normalises entities to ontology IDs:

$$
E(u_j) = LLM_E(u_j, P_E) \odot D_E
$$
($\odot D_E$: dictionary/ontology filtering to filter out false positives and normalizes entity mentions to canonical)

| Raw Mention      | Normalised Form | Type           |
| ---------------- | --------------- | -------------- |
| MTX-531          | MESH:C000012345 | Drug           |
| HNSCC            | UMLS:C0007107   | Disease        |
| PIK3CA mutations | MESH:D011061    | Gene           |
| PDX models       | —               | Research Model |
Compare the normalised entity against the KG nodes to know if it appears in the KG or not by using cosine similarity in the vector space. Any entity whose embedding distance to all known KG nodes exceeds threshold $\rho$ is flagged as **new** and added to candidate set V⁺.
$$
\hat{e} = \operatorname*{arg\,min}_{v \in V} d(\phi(e), \psi(v)),
$$
$$

$$

---
## 3.5. Step 5: Relationship Extraction Agent (REA)

For each entity pair in the summary, predicts relation probabilities:

$$
p(r | ê_i, ê_j, u_j) = LLM_R(ê_i, ê_j, u_j, P_R)
$$

Multi-label prediction is allowed (a single passage can imply multiple relations). For our example, any relation `r` with p(r) ≥ $\theta$ ᵣ is selected:

```
(MTX-531, treats, HNSCC)          p = 0.99  ✓
(MTX-531, targets, PIK3CA mutations) p = 0.85  ✓
(MTX-531, used_in, PDX)            p = 0.75  ✓
```

---

## 3.6. Step 6: Schema Alignment Agent (SAA)

Checks whether each new entity/relation type exists in the KG schema by asking LLM to estimate the possibility an entity/relation belongs to type $\uptau$:
$$
\uptau * = \operatorname*{arg\,max}_{\uptau \in T} LLM_{SAA}(v, \uptau, P_{align})
$$
If type exists -> check the current constraint, otherwise -> new entities/relations type

>"MTX-531" is new -> SAA classifies it as type `Drug` - Already exist type. 
>The relation "targets" is new -> SAA maps it to the existing KG relation `inhibits` as the closest semantic match.
>`(Drug, inhibits, Disease)` satisfies previous schemas



---
## 3.7. Step 7: Conflict Resolution Agent (CRA)

Checks for logical contradictions with existing KG edges. Using an LLM-based debate prompt to check if triplet contradicts with old ones:

$$
LLM_{CRA}(t_{new}, t') → {Agree, Contradict, Ambiguous}
$$

> Suppose the existing KG already contains `(MTX-531, causes, HNSCC)`. The CRA detects a direct contradiction with the new `(MTX-531, treats, HNSCC)` and discarded / marks for **expert review** based on confidence
---

## 3.8. Step 8: Evaluator Agent (EA)

Aggregates three scores for each surviving triplet using sigmoid-weighted verification signals:

```
C(t)  = σ(Σ αᵢ vᵢ(t))    [Confidence]
Cl(t) = σ(Σ βⱼ cⱼ(t))    [Clarity]
R(t)  = σ(Σ γₖ rₖ(t))    [Relevance]

integrate(t) = 1  iff  [C(t) + Cl(t) + R(t)] / 3 ≥ Θ
```

> For `(MTX-531, treats, HNSCC)`: C = 0.99, Cl = 0.50, R = 0.80 -> mean = 0.76 ≥ $\Theta$ -> **integrated** into the KG.

## 3.9. Agent Summary Table

|Agent|Input|Output|Key Role|
|---|---|---|---|
|Ingestion (IA)|Raw PDF|Normalised JSON|Format cleaning, metadata|
|Reader (RA)|Normalised text|Scored segments|Relevance filtering|
|Summarizer (SA)|Segments|Concise summaries|Noise reduction|
|Entity Extraction (EEA)|Summaries|Normalised entities|NER + ontology linking|
|Relationship Extraction (REA)|Entity pairs + text|Candidate triplets|Relation classification|
|Schema Alignment (SAA)|New entities/relations|Type assignments|Ontology consistency|
|Conflict Resolution (CRA)|New + existing triplets|Agree/Contradict/Ambiguous|Logical consistency|
|Evaluator (EA)|Verified triplets|Integration decision|Final quality gate|

---

# 4. Contributions

- **Multi-Agent KG Enrichment Framework**: The first end-to-end, hierarchically organised, nine-agent LLM with domain-adaptive prompting pipeline for KG enrichment. 

- **Multi-Faceted Evaluation Protocol** In the absence of gold-standard biomedical KG benchmarks for enrichment, the authors design a complementary evaluation suite covering structural metrics (ΔCov, ΔCon), correctness (RLC, RHE), and downstream utility (CQA).

---

# 5. Gaps

- **Over-Reliance on LLM-Based Evaluation** The primary correctness metric (RLC) is judged by another LLM (DeepSeek-v3). The human evaluation (RHE) is limited to two domain experts and is not reported with inter-annotator agreement scores.

- **No Gold-Standard Benchmark** KARMA is evaluated only on a curated PubMed subset with no comparison against an established held-out KG completion benchmark (e.g., FB15k-237, BioKG). This makes it difficult to objectively position results relative to the broader literature and weakens reproducibility claims.

- **Single Backbone Per Experiment** All agents within a single experimental run share the same LLM backbone. Given that different backbones have different strengths, a mixed-backbone configuration could outperform any single backbone.

- **Schema Alignment on Dynamic Ontologies** The SAA assumes a relatively static KG schema (Disease, Drug, Gene, ...). In fast-moving fields, entirely new entity types and relation classes emerge (e.g., "long COVID phenotype", "liquid biopsy marker"). The framework has no mechanism for proposing, validating, and integrating new schema classes automatically.

- **Evaluation Limited to Biomedical Domains** All experiments are conducted on PubMed biomedical articles. The paper claims broad applicability ("healthcare, finance, autonomous systems") but provides no cross-domain experiments.