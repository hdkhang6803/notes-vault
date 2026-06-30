---
Date: "2026"
Authors: Vitor Gaboardi dos Santos, Boualem Benatallah, Silvana Togneri MacMahon
Venue: CAiSE
Paper:
Memory type:
Agent env:
Record format:
Memory architecture:
Tackle Module: Benchmark Creation
Need offline initialization: false
Fine-tuning?: false
Other tags:
Code: https://github.com/vitorgaboardi/ConstraintAPIBench
---
### 1. Terminology

|Term|Definition|
|---|---|
|**OpenAPI Specification (OAS)**|A machine-readable format for describing RESTful APIs, including endpoints, parameters, constraints, and descriptions|
|**Utterance**|A natural language user statement expressing an intent to invoke an API call|
|**Utterance–API call pair**|A paired dataset record linking a natural language utterance to a valid structured API call|
|**Constraint-Aware Pipeline (CAP)**|The proposed 3-step generation pipeline that extracts, models, and enforces API constraints during utterance–API call synthesis|
|**Description-Based Pipeline (DBP)**|The baseline pipeline from Sheng et al. [10] that uses LLM prompting with naturalness and parameter coverage guidelines but no explicit constraint modeling|
|**OAS Enrichment**|CAP's process of augmenting the original OAS with a "constraint" field per parameter and removing technical parameters before generation|
|**Values constraint (VAL)**|Constraint capturing the permissible range or enumerated set of values for a parameter|
|**Format constraint (FOR)**|Constraint specifying the required string format for a parameter value (e.g., IATA codes, ISO currency codes, date formats)|
|**ID constraint**|Boolean flag indicating whether a parameter represents an identifier not naturally known to end users|
|**Technical constraint (TEC)**|Boolean flag indicating whether a parameter is intended for developers rather than end users (e.g., API key, page offset, access token)|
|**Interdependency constraint (IDP)**|Logical dependencies between parameters (requires, or, one-of, all-or-none, zero-or-one, arithmetic)|
|**Parameter Coverage (PC)**|Proportion of an API method's parameters exercised across all generated utterances|
|**Parameter Combination Coverage (PCC)**|Count of unique parameter combinations across generated API calls, reflecting intent diversity|
|**Naturalness**|Whether a generated utterance sounds like an authentic human request to a chatbot|
|**Constraint adherence**|Whether generated API call parameter values comply with OAS-specified constraints|
|**LLM-as-a-judge**|Using an LLM (here: Llama-3.1-70B and GPT-4.1) to evaluate text quality attributes as a proxy for human annotation|

---

### 2. Summary

This paper propose**s CAP (Constraint-Aware Pipeline),** a three-step framework for generating synthetic utterance–API call pair datasets from OpenAPI Specifications. CAP addresses three quality dimensions overlooked by prior work: **naturalness, parameter diversity, and constraint adherence.** The pipeline first extracts five constraint types from OAS, enriches the specification with explicit constraint annotations while filtering developer-only parameters, and then prompts LLMs with structured rules governing all three quality dimensions. Evaluated across 1,044 API methods using GPT-4o and DeepSeek-V3 and compared against the DBP baseline, CAP consistently improved naturalness (by 16–23%), parameter coverage (by 6–8 percentage points), and constraint adherence (reducing violations by 42–59%).

---

### 3. Why

**Problem 1 — Dataset quality gaps in utterance–API call generation.** Prior pipelines such as ToolBench and DBP (Sheng et al.) overlook key quality attributes simultaneously. They generate utterances that are unnatural (e.g., referencing internal IDs users would not know), fail to cover the full parameter space, or produce API calls that violate OAS-specified constraints --> degrade downstream performance of tool-using LLMs trained on such data.

**Problem 2 — Constraint violations produce infeasible API calls.** Existing approaches do not explicitly model inter-parameter dependencies and value/format constraints from OAS. This leads to generated API calls with invalid parameter values or incompatible parameter combinations, making the data unsuitable for training reliable API-grounded language interfaces.

---

### 4. How

#### 4.1 Step 1 — API Parameter Constraint Extraction

_"What constraints does this API parameter carry?"_

CAP prompts an LLM to extract five constraint types from the OAS for each parameter. The LLM receives the API name, API description, method name, method description, and the full parameter list (name, type, description, default, required flag). A few-shot example using a flight search API with ten parameters illustrates at least one instance of each constraint type.

The five extracted types are:

- **VAL** — min/max numeric bounds or an enumerated closed set of values
- **FOR** — string format requirements (e.g., `YYYY-MM-DD`, IATA airport code `MAD`)
- **ID** — boolean; True if the parameter is an identifier not naturally known to users
- **TEC** — boolean; True if the parameter is developer-facing (e.g., `page_size`, `api_key`, `offset`); False for user-meaningful sorting or limit params
- **IDP** — six inter-parameter dependency types from Martin-Lopez et al. [5]: requires, or, one-of, all-or-none, zero-or-one, arithmetic

> **Example:** For a flight search method, extraction might yield: `departureDate` has `FOR = YYYY-MM-DD`, `VAL = min: 2022-01-01`; `origin` has `FOR = IATA code`; `maxStops` has `ID = False, TEC = False`, `VAL = min: 0`; `apiKey` has `TEC = True`.

> **Design rationale:** The `TEC` constraint is novel to this work — it prevents developer-only parameters from appearing in utterances, which is a key driver of unnaturalness in prior pipelines.

#### 4.2 Step 2 — OAS Enrichment

_"How do we make constraints available to the generator?"_

The original OAS is enriched in two ways before generation:

1. A `"constraint"` field is appended to each parameter's entry, listing all constraints extracted in Step 1.
2. Parameters where `TEC = True` are **removed** from the enriched OAS entirely.

> **Example (continued):** In the enriched OAS, `departureDate` now explicitly carries `"constraint": {"format": "YYYY-MM-DD", "min": "2022-01-01"}`. The `apiKey` parameter entry is removed.

> **Design rationale:** Explicit constraint fields give the generation LLM a structured, unambiguous reference — reducing the need to infer constraints from prose descriptions, which are often incomplete or absent.

#### 4.3 Step 3 — Utterance–API Call Pairs Generation

_"How do we enforce all three quality dimensions during generation?"_

CAP prompts LLMs with the enriched OAS and a structured rule set across three categories:

**Constraint Adherence Rules (4 rules):**

1. Required parameters must appear in every generated pair
2. Values must respect numeric ranges and enumerated sets (VAL)
3. Values must conform to format constraints (FOR)
4. Parameter combinations must satisfy all interdependency constraints (IDP)

**Naturalness Rules (5 rules):**

1. Omit the API name unless it is a consumer brand (e.g., Spotify, YouTube)
2. No generic placeholders (e.g., "this URL", "this name")
3. Every API call parameter value must be inferable from the utterance
4. Avoid user-unnatural formats in utterances (e.g., say "Madrid", not lat/lon coordinates)
5. If `ID = True`, do not generate artificial IDs; use the real-world entity the ID refers to

**Parameter Diversity Rules (4 rules):**

1. Every optional parameter must appear at least once across the batch of generated utterances
2. Use different combinations of optional parameters across utterances
3. Cycle through enumerated values across utterances
4. Avoid repeating the same parameter values across utterances

> **Example (continued):** A DBP-generated utterance might read: _"Find similar songs to the one identified by key 987654321"_ — violating naturalness by using an internal ID. CAP, applying rule 5 and the ID constraint, instead generates: _"Can you suggest some songs similar to 'Shape of You' by Ed Sheeran?"_

---

### 5. Benchmarks

#### 5.1 Baselines

|Baseline|Description|
|---|---|
|**Description-Based Pipeline (DBP)**|Sheng et al. [10]; prompts LLMs with guidelines for naturalness, parameter coverage, and value inclusion — but without explicit OAS constraint modeling. Used as the primary comparison.|

#### 5.2 Datasets / Related Benchmarks

- [[ConstraintAPIBench]] — Dataset released by this paper; 237 APIs, 1,044 methods, 10 utterance–API call pairs per method, generated under both CAP and DBP with GPT-4o and DeepSeek-V3
- [[ToolBench]] — Qin et al. [9]; source of the 237-API subset used here; covers 16,000+ real-world APIs from RapidAPI
- [[Gorilla]] — Patil et al. [8]; LLM-to-API alignment benchmark

#### 5.3 Results

**Naturalness** (% utterances judged natural; evaluated by Llama-3.1-70B and GPT-4.1 as judges):

|Model|Pipeline|NAT-LLA3.1|NAT-GPT4.1|Cohen's κ|
|---|---|---|---|---|
|GPT-4o|DBP|1,080 (50.7%)|991 (46.5%)|80.2%|
|GPT-4o|CAP|1,384 (67.1%)|1,293 (62.7%)|75.4%|
|DeepSeek-V3|DBP|967 (44.3%)|930 (42.6%)|80.5%|
|DeepSeek-V3|CAP|1,479 (67.5%)|1,526 (69.6%)|75.1%|

CAP improved naturalness by +16.4% (GPT-4o) and +23.1% (DeepSeek-V3) under NAT-LLA3.1. Human validation on 400 sampled utterances confirmed substantial inter-rater agreement with LLM judges (κ = 0.70 ± 0.07 vs. GPT-4.1; κ = 0.66 ± 0.06 vs. Llama-3.1-70B).

**Parameter Diversity** (PC = parameter coverage; PCC = unique parameter combinations; TP = total parameters):

|Model|Pipeline|PC|PCC|TP|
|---|---|---|---|---|
|GPT-4o|DBP|0.9092|373|592|
|GPT-4o|CAP|0.9823|416|545|
|DeepSeek-V3|DBP|0.9140|416|592|
|DeepSeek-V3|CAP|0.9733|376|499|

CAP improved PC for both models (+6.3 pp for GPT-4o; +5.9 pp for DeepSeek-V3). GPT-4o also gained in PCC (373 → 416). DeepSeek-V3's PCC dropped (416 → 376), attributed to filtering out 93 technical parameters which reduced the combinatorial space.

**Constraint Adherence** (violation counts; lower = better):

|Model|Pipeline|VAL|FOR|IDP|Total|
|---|---|---|---|---|---|
|GPT-4o|DBP|10|49|7|66|
|GPT-4o|CAP|2|21|4|27|
|DeepSeek-V3|DBP|26|59|6|91|
|DeepSeek-V3|CAP|19|34|0|53|

CAP reduced total violations by 59% for GPT-4o (66 → 27) and 42% for DeepSeek-V3 (91 → 53). The largest gains were in FOR violations. CAP eliminated all IDP violations for DeepSeek-V3.

---

### 6. Strengths

- **Novel TEC constraint type:** Introducing the Technical constraint to filter developer-facing parameters is a principled, practically impactful contribution — directly addressing a root cause of unnaturalness that prior work overlooked.
- **Holistic quality framework:** CAP simultaneously targets three quality dimensions (naturalness, parameter diversity, constraint adherence) rather than optimising a single metric, aligning better with the real requirements for training robust tool-using LLMs.
- **Empirically validated LLM-as-judge pipeline:** Human annotation on 400 utterances confirmed substantial agreement with the two LLM judges, lending credibility to the automated naturalness evaluation at scale.
- **Cross-model generalisability:** Results are consistent across two architecturally distinct frontier LLMs (GPT-4o and DeepSeek-V3), suggesting CAP's structured prompting approach is model-agnostic rather than tuned to one system.

---

### 7. Gaps

**Constraint taxonomy tied to OAS/REST APIs.** The five constraint types are grounded in the OpenAPI Specification and REST API testing literature. Agentic frameworks increasingly use non-REST tool definitions (e.g., function-calling schemas in LangChain, MCP tool descriptors, GraphQL). CAP's extraction and enrichment logic does not transfer directly to these formats.  
=> Future work should generalise the constraint taxonomy and extraction step to cover function-call schema formats used in modern agentic systems, enabling CAP-style generation for multi-step tool-use pipelines.

**PCC regression for DeepSeek-V3.** Filtering 93 technical parameters from the parameter pool reduced PCC for DeepSeek-V3 from 416 to 376, meaning fewer unique parameter combinations were generated. The current diversity rules do not compensate for a reduced combinatorial space.  
=> A combinatorial sampling strategy (e.g., covering arrays or pairwise testing methods) could ensure diversity targets are met independently of the number of available parameters.

**Evaluation limited to constraint violation counting, not downstream task performance.** The paper measures constraint adherence via automated rule checking rather than assessing whether CAP-generated data actually improves model fine-tuning or API-calling accuracy on held-out benchmarks.  
=> CAP-generated datasets should be used to fine-tune LLMs and evaluated on API-calling benchmarks (e.g., ToolBench evaluation, ToolLLM's test sets) to establish the downstream utility of higher-quality synthetic data.

---

### 8. Highlights