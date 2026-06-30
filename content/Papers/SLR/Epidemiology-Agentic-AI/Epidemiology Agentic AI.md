---
Tags:
Date: "2026"
Authors: Shreyansh Padarha, Ryan Othniel Kearns, Tristan Naidoo, Lingyi Yang, Łukasz Borchmann, Piotr BŁaszczyk, Christian Morgenstern, Ruth McCabe, Sangeeta Bhatia, Philip H. Torr, Jakob Foerster, Scott A. Hale, Thomas Rawson, Anne Cori, Elizaveta Semenova, Adam Mahdi
Venue: ICML
Paper: Evaluating AI-based Scientific Knowledge Synthesis with Epidemiological Systematic Reviews
---
# 1. Terminology

| Term                                      | Definition                                                                                                                                |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| SLR (Systematic Literature Review)        | A comprehensive synthesis requiring retrieval, screening, extraction, and analysis of scientific articles following a structured protocol |
| LRM (Language Reasoning Model)            | Large language model with inference-time scaling capability, used without fine-tuning                                                     |
| LLM (Large Language Model)                | Foundation model used for text understanding and generation tasks                                                                         |
| Agentic pipeline                          | A multi-stage AI system where models autonomously call tools, make decisions, and execute structured workflows                            |
| PERG (Pathogen Epidemiology Review Group) | Expert group conducting SLRs on WHO-designated priority pathogens; ground truth source                                                    |
| WHO priority pathogen                     | Pathogen designated by WHO as having high epidemic/pandemic potential requiring urgent preparedness                                       |
| OCR (Optical Character Recognition)       | Image-to-text conversion used to extract content from PDF pages                                                                           |
| Presence flagging                         | Sub-task that identifies whether an article contains a relevant data category (parameter, model, outbreak)                                |
| Bipartite matching                        | Optimal one-to-one correspondence algorithm used to align extracted records to ground truth for evaluation                                |
| REDCap                                    | Web-based data capture system used by PERG for structured extraction                                                                      |
| ScreenPrompt                              | Prompting methodology structuring screening into five components: objectives, criteria, chain-of-thought, abstract, structured output     |
| Living systematic review                  | Continuously updated review that incorporates new evidence as it emerges                                                                  |
| CFR (Case Fatality Rate/Ratio)            | Proportion of diagnosed cases resulting in death                                                                                          |
| R₀                                        | Basic reproduction number: average secondary cases from one case in a fully susceptible population                                        |
| Serial interval                           | Time between symptom onset in a primary and secondary case                                                                                |
| Incubation period                         | Time from infection to symptom onset                                                                                                      |
| Overdispersion (k)                        | Measure of individual variation in infectiousness                                                                                         |
| IFR (Infection Fatality Rate/Ratio)       | Proportion of all infections (including asymptomatic) resulting in death                                                                  |
| Seroprevalence                            | Proportion of a population with detectable antibodies indicating past or recent infection                                                 |
| Compartmental model                       | Mathematical model partitioning a population into disease states (e.g. SIR, SEIR)                                                         |
| Branching process                         | Stochastic model describing probabilistic offspring distributions for transmission chains                                                 |
| Theoretical model                         | Model using parameters from literature or arbitrary values, not fitted to actual data                                                     |
| vllm                                      | Open-source library for efficient LLM inference; used to self-host open-weight models                                                     |
| OpenRouter                                | API aggregation service providing access to multiple LLMs with per-token pricing                                                          |
| Rule of three                             | PERG aggregation rule: when ≥3 disaggregations of a parameter exist, extract as a range                                                   |
| Jaccard similarity                        | Set-based similarity metric J(A,B)=∥A∩B∥/∥A∪B∥J(A,B) = \\                                                                                 |

---

### 2. Summary

AgentSLR is an open-source, fully automated pipeline for systematic literature reviews in infectious disease epidemiology, targeting nine WHO priority pathogens. The pipeline chains six stages — article retrieval, abstract screening, PDF-to-Markdown conversion, full-text screening, structured data extraction, and report generation — using large language reasoning models with tool calling and schema validation throughout. Evaluated against expert-curated PERG ground truth across four pathogens (Ebola, Lassa, SARS-CoV-1, Zika), AgentSLR achieves human-comparable output quality while reducing review time from ~385 labour hours to ~20 wall-clock hours (58× calendar-time speed-up). A five-model ablation shows no single frontier LRM dominates across stages, and that the smallest tested model (gpt-oss-120b) achieves competitive performance at over 96× lower cost than the flagship closed-source alternative.

---

### 3. Why

**Scalability bottleneck in evidence synthesis.** Traditional SLRs take an average of 67 weeks and $141,000 in labour per review. As literature volume grows faster than reviewer capacity — particularly for emerging pathogens — the rate of evidence synthesis cannot keep pace with the rate of evidence production, creating critical gaps in outbreak preparedness.

**Compounding failure risk in multi-stage agentic pipelines.** Prior work shows LLMs can assist individual SLR stages (screening, extraction) in isolation, but errors propagate and compound across stages in end-to-end pipelines. No prior system had been validated against expert-curated annotations across all stages simultaneously, without LLM-as-a-judge, and with open-weight model compatibility — leaving the feasibility of trustworthy full-pipeline automation unestablished.

---

### 4. How

#### 4.1 Article Search and Retrieval — _"How do we acquire a comprehensive, deduplicated corpus?"_

AgentSLR queries three bibliographic databases (OpenAlex, PubMed, Europe PMC) using domain-specific Boolean search strategies covering seven core epidemiological domains. Retrieved records are deduplicated via a hierarchical five-level strategy (DOI → PMID → PMCID → OpenAlex ID → title-year combination). Full texts are retrieved from open-access sources through a cascading four-source pipeline (OpenAlex → Europe PMC → Unpaywall → OpenAlex DOI lookup), with parallel execution (16 workers), streaming downloads, magic-byte validation, and checkpointing.

> **Running example:** For MERS-CoV, the query uses expanded identifiers including spelling variants ("Middle East Respiratory Syndrome Coronavirus") and retrieves 23,204 articles; after cascading retrieval, PDFs are validated and cached, producing a clean corpus for downstream stages.
> 
> > [!note] **Design rationale:** The five-level deduplication hierarchy ensures that records appearing across multiple databases are collapsed into single entries without losing any identifier, while the cascading retrieval strategy maximises open-access coverage without redundant API calls.

---

#### 4.2 Title and Abstract Screening — _"How do we filter irrelevant articles at scale without fine-tuning?"_

Screening is framed as binary classification (include/exclude) using LRMs with the ScreenPrompt structure: study objectives, inclusion/exclusion criteria, chain-of-thought instructions, abstract content, and structured output format. The model reasons step-by-step against each criterion and outputs a final `<decision>` tag. The emphasis is on broad recall — uncertain cases are included — to avoid missing evidence recoverable at full-text stage.

Recall=TPTP+FN,Precision=TPTP+FP,F1=2⋅P⋅RP+R\text{Recall} = \frac{TP}{TP + FN}, \quad \text{Precision} = \frac{TP}{TP + FP}, \quad F_1 = \frac{2 \cdot P \cdot R}{P + R}Recall=TP+FNTP​,Precision=TP+FPTP​,F1​=P+R2⋅P⋅R​

> **Running example:** For a Zika article reporting R₀ estimates from mosquito transmission models, the screener checks whether it meets inclusion criterion 5(c) (reproduction number estimates) and is not a conference abstract (exclusion criterion 3). Chain-of-thought reasoning makes this judgment transparent and auditable.
> 
> > [!note] **Design rationale:** LRMs are preferred over fine-tuned classifiers because inference-time scaling allows the model to handle the heterogeneity of epidemiological abstracts without requiring labelled training data from each pathogen's literature.

---

#### 4.3 PDF-to-Markdown Conversion — _"How do we preserve document structure across complex scientific PDFs?"_

Each downloaded PDF is rendered page-by-page into high-resolution images, then processed by `mistral-ocr-2512` to recover text while preserving document hierarchy, LaTeX equations, and HTML tables. Parallel execution with 14 concurrent requests reduces per-document time to ~1.1 seconds. The output is one Markdown file per article, consumed by all downstream stages.

> **Running example:** A SARS paper containing an SEIR model with LaTeX-formatted differential equations and an embedded parameter table is rendered such that both the equations and table structure are preserved in Markdown, allowing downstream extraction to correctly identify model type and parameter values.
> 
> > [!note] **Design rationale:** OCR-based rendering rather than direct PDF text extraction is used because many epidemiological papers are scanned or contain complex mixed-content layouts; page-image rendering handles these uniformly regardless of PDF encoding.

---

#### 4.4 Full-text Screening — _"How do we enforce stricter extractability criteria on the full article?"_

Full-text screening uses the same ScreenPrompt structure as abstract screening but applies stricter criteria: articles must contain extractable quantitative epidemiological parameters. Additional exclusion criteria are applied: literature reviews, meta-analyses, and case studies with fewer than 10 infected individuals are excluded. The model is instructed to critically evaluate whether the article contains something extractable, not merely on-topic.

> **Running example:** An Ebola paper that describes outbreak dynamics qualitatively but reports no quantitative transmission estimates would pass abstract screening (outbreak descriptions mentioned) but be excluded at full-text screening for failing criterion 6 (no extractable quantitative parameters or models).
> 
> > [!note] **Design rationale:** Separating abstract and full-text screening allows the pipeline to process abstracts cheaply at scale and reserve the more expensive full-text pass for a smaller candidate set, while the stricter full-text criterion reduces false positives entering the costly extraction stage.

---

#### 4.5 Data Extraction — _"How do we extract structured epidemiological data reliably from heterogeneous articles?"_

Extraction targets three categories — epidemiological parameters, transmission models, and outbreaks — using a multi-stage, schema-constrained tool-calling framework. For each category, the pipeline first runs **presence flagging**, then **targeted extraction** via category-specific tool calls with JSON schema validation. Invalid outputs are rejected and the model is prompted to correct them before proceeding.

**For parameters**, a five-step workflow is used:

s(E,E^)=∑k∈Fwk⋅J(E[k],E^[k]),J(A,B)=∣A∩B∣∣A∪B∣s(E, \hat{E}) = \sum_{k \in F} w_k \cdot J(E[k], \hat{E}[k]), \quad J(A,B) = \frac{|A \cap B|}{|A \cup B|}s(E,E^)=k∈F∑​wk​⋅J(E[k],E^[k]),J(A,B)=∣A∪B∣∣A∩B∣​

Steps: (1) class-level screening with quotation extraction, (2) value extraction per class with class-specific schemas, (3) population context extraction (sex, age, location, sample type), (4) uncertainty extraction (value type, statistical approach, bounds), (5) aggregation following PERG's rule of three.

|Parameter Class|Tool Schema Fields (selected)|
|---|---|
|Reproduction number|`value`, `type` (R₀/Re), `transmission`, `method`|
|Severity (CFR/IFR)|`value`, `numerator`, `denominator`, `parameter_type`, `method` (naive/adjusted)|
|Human delay|`value`, `delay_type` (serial interval, incubation period, etc.), `unit`|
|Seroprevalence|`value` (proportion), `parameter_type` (IgG/IgM/PRNT/etc.), `numerator`, `denominator`|
|Risk factors|`name` (list), `outcome` (list), `significant`, `adjusted`|
|Overdispersion|`value`, `unit`|
|Attack rate|`value`, `unit` (percentage/rate), `type` (primary/secondary)|
|Growth rate|`value`, `unit` (per day/week/etc.)|
|Mutation rate|`value`, `type` (substitution/evolutionary/mutation rate), `genome_site`|

**For models**, a two-stage workflow: binary screening (include/exclude mechanistic transmission models) followed by iterative `extract_model_data` tool calls, one per distinct model. Schema enforces controlled vocabularies for model type, compartmental architecture, stochasticity, transmission routes, assumptions, and interventions.

**For outbreaks**, binary screening identifies concluded outbreak events with defined case counts, then `extract_outbreak_data` is called once per distinct outbreak (distinguished by location and/or time). All values are extracted as stated in the paper — no calculation or inference of missing values is permitted.

> **Running example:** For a Lassa fever paper reporting CFR = 0.23 (deaths = 46, cases = 201) in hospitalised patients in Nigeria during a 2018 outbreak:
> 
> - **Flagging:** severity and outbreak classes flagged as present.
> - **Value extraction:** `value=0.23`, `parameter_type=CFR`, `method=naive`, `numerator=46`, `denominator=201`.
> - **Population extraction:** `population_sample_type=hospital_based`, `population_countries=["Nigeria"]`, `method_moment_value=mid_outbreak`.
> - **Outbreak extraction:** `outbreak_country="Nigeria"`, `cases_confirmed=201`, `deaths=46`, `outbreak_start_year=2018`.
> 
> > [!note] **Design rationale:** Tool calling with schema-level JSON validation mimics the structure of PERG's REDCap survey forms, enforcing field constraints at the point of generation rather than through post-hoc cleaning. Separating flagging from extraction allows the model to cast a wide net first, then apply precision when generating structured records — matching the asymmetric precision-recall profile observed in results.

---

#### 4.6 Report Generation — _"How do we synthesise extracted data into a trustworthy, evidence-grounded narrative?"_

Extracted data are converted into living reviews through a two-phase process. In the **deterministic assembly phase**, descriptive statistics, figures, and evidence tables are computed programmatically and compiled into a Markdown draft alongside a content manifest. In the **LRM self-refinement phase**, an iterative critique–revise loop refines the narrative:

W(k)=revise(W(k−1),C(k)),C(k)=critique(W(k−1))W^{(k)} = \text{revise}(W^{(k-1)}, C^{(k)}), \quad C^{(k)} = \text{critique}(W^{(k-1)})W(k)=revise(W(k−1),C(k)),C(k)=critique(W(k−1))

Critique uses an 8-dimension rubric (data fidelity, figure/table presence, traceability, clarity, completeness, interpretation blocks, formatting). Interpretation is permitted only inside explicitly labelled `> AI-Interpretation:` blockquotes; all other claims must cite figures, tables, or dataset statistics. Non-negotiable asset checks enforce that all required figures and tables appear in the final output.

> **Running example:** For Ebola, the pipeline assembles 1,104 outbreak records into a surveillance review, generates temporal distribution and geographic burden figures, then runs the self-refinement loop. If the critique flags a claim about case counts that lacks a `(Table Y)` citation, the revision step corrects it. The final report labels all interpretive statements explicitly as AI-generated.
> 
> > [!note] **Design rationale:** Separating deterministic assembly from LRM synthesis ensures that figures and quantitative summaries are never hallucinated — the LRM can only narrate pre-computed artefacts. The rubric-based critique loop is used instead of single-pass generation because iterative refinement improves attribution and completeness on long, evidence-heavy documents.

---

### 5. Benchmarks

#### 5.1 Baselines

|System|Approach|Evaluation|Limitation vs AgentSLR|
|---|---|---|---|
|otto-SR (Cao et al., 2025a)|LLM extraction with post-hoc human correction|LLM-as-a-judge with corrected labels|Closed source; no open-weight support; no stage-level evaluation|
|A4SLR|Prompt-based with domain templates|Human reference|No open-weight support; no stage-level evaluation|
|MetaBeeAI (Parkinson et al., 2025)|Multi-pass chunk retrieval|Human reference + independent eval|Closed source; no stage-level evaluation|
|ScreenPrompt (Cao et al., 2025b)|Prompt templates for screening only|Multiple SLRs|Screening stage only; no extraction|
|Human (PERG)|Full manual SLR workflow|Ground truth source|~385 hours per SLR; ~$141K per review|

#### 5.2 Benchmarks / Ground Truth Datasets

- [[epireview R package]] — PERG-curated extraction data for priority pathogens; used as ground truth for all evaluation stages
- [[priority-pathogen R package]] — Structured parameter, model, and outbreak annotations for Ebola, Lassa, SARS-CoV-1, Zika
- [[OpenAlex]] — Bibliographic database; primary source for article retrieval and PDF URLs
- [[PubMed]] — Bibliographic database; secondary retrieval source
- [[Europe PMC]] — Bibliographic database; tertiary retrieval source with full-text availability metadata
- [[Unpaywall API]] — Open-access PDF resolution; tertiary PDF retrieval source
- [[NCBI PMC ID Converter API]] — Identifier cross-referencing for PDF retrieval
- [[REDCap (PERG)]] — Web-based survey used by PERG human annotators as extraction reference schema

#### 5.3 Results by Stage

**Article Screening (7 pathogens: Marburg, Ebola, Lassa, SARS, Zika, MERS, Nipah)**

_Two-stage AI pipeline (abstract → full-text):_

- Overall recall: 0.81, precision: 0.75, F₁: 0.77
- Best pathogen: Nipah (F₁ 0.85); worst: Zika (F₁ 0.69)

_Human abstract → AI full-text:_

- Overall recall: 0.92, F₁: 0.87 — highest overall F₁ configuration
- Full-text screening 118× faster than human reviewers (2.0 s/article vs. 4 min/article)

_Direct AI full-text:_

- Recall: 0.89 (improved over two-stage), precision: 0.68 (reduced); 2.3× slower than two-stage

**Data Extraction (4 pathogens: Ebola, Lassa, SARS, Zika — gpt-oss-120b)**

_Parameters:_

- Flagging: P=0.51, R=0.92, F₁=0.66
- Count: P=0.83, R=0.47, F₁=0.59
- Extraction: P=0.52, R=0.57, F₁=0.54
- Near-perfect accuracy for `method` and `single_type_uncertainty`; weakest on `value` and population context fields

_Models:_

- Flagging: P=0.90, R=0.91, F₁=0.91
- Count: P=0.52, R=0.99, F₁=0.68
- Extraction: P=0.63, R=0.74, F₁=0.67
- Strong on `model_type`, `stoch_deter`, `code_available`; weakest on `assumptions`, `interventions`, `transmission_route`

_Outbreaks (Lassa + Zika only):_

- Flagging: P=0.63, R=0.76, F₁=0.61
- Count: P=0.66, R=0.72, F₁=0.69 (high variance ±0.28 for recall)
- Extraction: P=0.85, R=0.76, F₁=0.79 — highest precision among all data types
- Perfect country identification (1.00/1.00); weakest on specific location and pre-outbreak status

**Human Expert Validation (6 epidemiologists)**

|Data Type|Flagging Precision|Extraction Accuracy|Mean Competence (1–7)|
|---|---|---|---|
|Parameters|0.66|0.77|4.2|
|Models|0.40|0.83|2.8|
|Outbreaks|0.61|0.80|3.9|

Threshold for "useful under human supervision" = 4.0; parameters and outbreaks meet or exceed this threshold.

**Model Ablation (5 models across all stages)**

|Model|Avg F₁|Cost/SLR|Notes|
|---|---|---|---|
|gpt-oss-120b|0.70|$13.9|Best value; leads full-text screening|
|Kimi-K2.5|0.74|$277|Best overall F₁; leads abstract screening and parameter extraction|
|GLM-4.7|0.73|$811|Best model extraction; 358B parameters|
|DeepSeek-V3.2|0.67|$73.6|Most variable; weakest at screening, competitive at extraction with function calling|
|GPT-5.2|0.69|$1,348|Highest cost; best outbreak extraction; 91.1K output tokens/article at parameter extraction|

No single model dominates across all stages. Higher cost does not predict higher performance.

**Runtime Comparison**

|Stage|Human (hours)|AgentSLR (hours)|Speed-up|
|---|---|---|---|
|Abstract screening|114.2|1.6|71×|
|PDF-to-Markdown|0|2.8|—|
|Full-text screening|73.5|0.62|118×|
|Data extraction|197.5|13.4|15×|
|**Total**|**385.1**|**20.0**|**19.3× (58× calendar)**|

---

### 6. Strengths

- **End-to-end open-source pipeline with stage-isolated evaluation.** AgentSLR is the only system satisfying all five key properties simultaneously: open code, open-weight compatibility, human reference ground truth, independent evaluation (no LLM-as-a-judge), and stage-level metrics enabling failure attribution. This combination makes results reproducible and directly improvable at the component level.
- **Substantial and demonstrated efficiency gains.** The 58× calendar-time speed-up and 19.3× labour-hour reduction are measured against a real expert workflow (PERG) on a real high-stakes domain, not synthetic benchmarks. Full-text screening in particular runs 118× faster than human reviewers, enabling the "living review" use case where literature is continuously re-ingested.
- **Cost-competitive open-weight model performance.** gpt-oss-120b achieves comparable overall F₁ (0.70) to the flagship closed-source GPT-5.2 (0.69) at over 96× lower cost ($13.9 vs $1,348 per SLR), with the added benefit of version pinning and local deployment — properties critical for reproducibility in long-running living reviews.
- **Ecological validity confirmed via closed-access comparison.** The open-access corpus (26.2% of PERG articles) is shown to be a non-systematically biased sample through a matched comparison against 1,004 closed-access articles, with all performance differences within overlapping 95% confidence intervals. This validates that results generalise to the broader literature.

---

### 7. Gaps

**Low parameter extraction precision at flagging stage.** AgentSLR achieves recall of 0.92 but precision of only 0.51 for parameter class flagging, meaning roughly half of flagged parameter classes per article are false positives that propagate into the extraction stage.  
=> Future work could introduce a lightweight re-ranking or confirmation step between flagging and extraction, or fine-tune a small classifier on PERG-labelled data to improve flagging precision without sacrificing recall.

**Weak population context extraction.** Population fields (sample type, population group, sex, age) are the most challenging extraction sub-group, with group accuracy of 0.59 in expert validation. Many valid options share overlapping descriptions in epidemiological literature (e.g. "persons under investigation" has a specific clinical/epidemiological definition an LLM may not apply consistently).  
=> Domain-adaptive fine-tuning on PERG REDCap extraction data, or retrieval-augmented prompting with field-level definitions, could substantially close this gap.

**Content filter refusals from closed-source providers.** Claude Opus 4.5 and Sonnet 4.5, and all Claude 4.0+ models, consistently refused to process epidemiological content, attributed to content filters misclassifying disease terminology as bioweapons-related. This rendered an entire closed-source model family unavailable for legitimate public-health research.  
=> API-level safe-use certifications or domain whitelisting mechanisms for vetted public-health applications could prevent systematic exclusion of capable models from critical research workflows.

**Open-access coverage limits ground truth comparability.** Only 26.2% of PERG articles are open-access, constraining both the training signal available and the ability to evaluate on the full PERG corpus. The gap is particularly pronounced for SARS-CoV-1 (16.0%).  
=> Future work integrating institutional access agreements or publisher partnerships for systematic review pipelines could expand evaluation coverage; federated evaluation where institutions run the pipeline locally on closed-access corpora is another avenue.

**No formal human uplift quantification.** The paper demonstrates that expert epidemiologists report improved efficiency with AgentSLR, but does not include a controlled human uplift study measuring actual time savings and error rates in a human-in-the-loop deployment.  
=> A randomised controlled trial comparing human-only vs. human+AgentSLR review time and quality across a new pathogen SLR would provide the causal evidence needed to justify operational adoption.

---

### 8. Highlights