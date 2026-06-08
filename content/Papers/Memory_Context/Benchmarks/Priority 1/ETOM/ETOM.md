---
Tags:
Date: "2026"
Authors: " Jia-Kai Dong, I-Wei Huang, Chun-Tin Wu, Yi-Tien Tsai"
Venue: EACL
Paper: "ETOM: A Five-Level Benchmark for Evaluating Tool Orchestration within the MCP Ecosystem"
---
# 1. Terminology

| Term                             | Definition                                                                                                                                                                                         |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **MCP (Model-Context Protocol)** | A federated architecture that organizes tools into semantically coherent, independently operating servers, shifting agent task from flat API namespace to hierarchical multi-server orchestration. |
| **Tool Orchestration**           | The agent's ability to select and chain multiple tools across servers to accomplish multi-step, goal-oriented tasks.                                                                               |
| **Equal Function Sets (EFS)**    | Groups of functionally equivalent tools across different servers that can achieve the same user intent, used to handle functional overlap in evaluation.                                           |
| **Functional Overlap**           | When multiple tools can fulfill the same user intent; a pervasive challenge in real-world tool orchestration.                                                                                      |
| **End-to-End Evaluation**        | Comprehensive assessment of both tool retrieval and LLM reasoning components within a complete orchestrator pipeline, not in isolation.                                                            |
| **Server Hierarchy**             | Multi-level organizational structure where tools are grouped by semantic domain (servers), creating a tree-like navigation space unlike flat tool namespaces.                                      |
| **Cross-Server Orchestration**   | Multi-hop tool chaining that spans multiple servers, requiring agents to understand server boundaries and dependencies.                                                                            |
| **Intra-Server Orchestration**   | Sequential tool chaining within a single server, focusing on data flow and dependency management within one domain.                                                                                |
| **Contextual Drift**             | Failure mode where agents lose critical context when transitioning across server boundaries during multi-server planning.                                                                          |
| **Out-of-Scope Detection**       | Agent's capability to recognize when user requests exceed available tool capabilities and appropriately reject them.                                                                               |
| **Capability Gap**               | Functional domain outside the digital MCP ecosystem (e.g., physical world interaction, real-time sensory processing).                                                                              |
| **Five-Level Curriculum**        | Structured progression from foundational single-tool retrieval (L1-L2), to sequential intra-server orchestration (L3), to complex cross-server planning (L4), to robustness checks (L5).           |
| **Pseudo Output Schemas**        | Inferred JSON schemas for tool outputs enabling logical task decomposition without requiring live API execution.                                                                                   |
| **Exact Match (EM)**             | Binary metric for single-step tasks: query successfully routed to correct tool or correctly rejected (L1, L2, L5).                                                                                 |
| **F1 Score**                     | Node-based metric for multi-step tasks: evaluates precision and recall of tool selection in execution plans (L3, L4).                                                                              |
| **Normalized Latency (N-Lat.)**  | Latency relative to MCP-Zero Meta-Llama baseline, enabling fair comparison across architectures and foundation models.                                                                             |
| **Rule-Based Expert**            | Closed-loop reference for ground truth generation in DAgger training, avoiding physical failures of embodied experts.                                                                              |
| **Retrieval Breadth**            | Number of candidate tools retrieved in initial search; trades off noise (L2 degrades) vs. coverage (L4 improves).                                                                                  |
| **HDBSCAN Clustering**           | Density-based clustering algorithm used to group tool embeddings for capability mapping and gap identification.                                                                                    |

---

# 2. Metadata

## 2.1. Benchmark Composition

|Property|Value|
|---|---|
|**Total Servers**|491 real MCP servers (sourced from glama.ai registry)|
|**Total Tools**|2,375 distinct tools across servers|
|**Total Tasks**|2,075 evaluation tasks|
|**Room Types**|4 (kitchen, bedroom, bathroom, living room) with 120 total scenes|
|**Object Classes**|73 classes, each with 1-10 instances per scene|
|**Task Types**|6 canonical types representing household/digital workflows|
|**Training Cost**|~$500 USD for full data generation pipeline using proprietary LLMs|
|**Data Privacy**|Full dataset not public; task generation pipelines and evaluation methodologies released for reproducibility|

## 2.2. Five-Level Curriculum Breakdown

|Level|Key Challenge|# Tasks|Avg. Plan Length|Avg. Servers|Metric|
|---|---|---|---|---|---|
|**L1: Direct Retrieval**|Foundational tool identification with explicit tool names|781|1.00|1.00|EM|
|**L2: Context-Aware Retrieval**|Disambiguation among functionally equivalent tools|773|1.00|1.00|EM against set|
|**L3: Intra-Server Chaining**|Sequential orchestration, data flow, dependency ordering|327|2.87|1.00|Node Set EM, F1|
|**L4: Cross-Server Chaining**|Multi-server orchestration, cross-domain planning|103|3.83|3.78|Node Set EM, F1|
|**L5: Robust Rejection**|Capability gap detection, out-of-scope rejection|91|0.00|0.00|Exact Rejection Match|

## 2.3. Equal Function Set Statistics

- **Initial Candidate Sets (Bottom-Up):** 145 sets based on semantic similarity
- **Final High-Confidence EFS (After Round-Trip Validation):** 95 sets
- **Refinement Process:** Bottom-up semantic discovery + top-down query-guided RAG + human platform-aware refinement
- **Validation Hours:** ~10 student-hours for manual borderline case examination

## 2.4. Server Category Distribution

- **30+ Functional Categories** spanning development tools, productivity apps, cloud platforms, data processing, messaging, and analytics
- **Real-World Representation:** Publicly available servers on glama.ai, reflecting current MCP ecosystem deployment trends
- **Platform Diversity:** Tools from GitHub, Heroku, Notion, HackMD, Semgrep, Stripe, and 480+ other services

---

# 3. What It Measures (What)

## 3.1. Task Definition

ETOM measures an agent's ability to orchestrate tools across a federated MCP ecosystem with increasing complexity, from single-tool retrieval to complex cross-server multi-hop workflows and robustness to out-of-scope requests.

### 3.1.1. Level 1: Explicit Single-Tool Retrieval

**Objective:** Establish baseline competence through direct tool invocation when platform and tool are explicitly mentioned.

**Example Task:**

- **Query:** "On Semgrep, run semgrep_scan on the file 'vulnerable_code.js' with the configuration file 'semgrep_config.yaml' to detect vulnerabilities and return findings in JSON format."
- **Expected Tool:** semgrep_scan from Semgrep MCP Server
- **Evaluation:** Exact match with tool name and server; no partial credit for similar tools
### 3.1.2. Level 2: Context-Aware Tool Retrieval (Disambiguation)

**Objective:** Assess reasoning under functional redundancy where multiple tools can fulfill the same request.

**Example Tasks:**
- **Query:** "Convert this image to JPEG format with 90% quality."
- **Expected Tools:** {convert_to_jpeg, image_format_converter}
- **Challenge:** Agents must recognize functional equivalence without preferring one over another
### 3.1.3. Level 3: Intra-Server Sequential Chaining

**Objective:** Measure agent's capacity to decompose multi-step goals into coherent execution plans within a single server.

**Example Task:**
- **Query:** "Analyze the code in src/main.py for security vulnerabilities, generate a detailed report, and send it to the development team."
- **Expected Execution Plan (DAG):**
    1. analyze_code (Semgrep) → scans Python file for security issues
    2. generate_report (Semgrep) → depends on output of step 1
    3. send_notification (Notification Server) → depends on step 2
- **Evaluation:** Exact sequence + tool selection accuracy + proper dependency ordering
- **Challenge:** Inferring data flow between tools without explicit schemas; maintaining coherent multi-step reasoning
### 3.1.4. Level 4: Cross-Server Compositional Chaining

**Objective:** Measure agent's capacity to orchestrate tools across multiple semantic domains with proper server boundary handling.

**Example Task:**
- **Query:** "Deploy the latest version of my web application to production, run security scans, and notify the team about the deployment status."
- **Expected Execution Plan (Multi-Server DAG):**
    1. deploy_app (Deployment Server) → deploys to production
    2. run_security_scan (Security Server) → depends on step 1
    3. send_notification (Notification Server) → depends on steps 1 AND 2
- **Contextual Requirements:** Intermediate results (deployment status, scan report) must flow across boundaries
- **Evaluation:** Exact sequence matching + cross-server dependency correctness + tool/server identification
### 3.1.5. Level 5: Robustness via Capability Gap Detection

**Objective:** Evaluate agent's ability to correctly reject requests that exceed available capabilities.

**Example Tasks:**

- **Query:** "Turn on the lights in my living room and adjust the temperature to 22°C."
- **Expected Response:** {server: "no", tool: "no"}
- **Reason:** Requires physical world interaction (smart home control) outside digital MCP ecosystem
- **Challenge:** Comprehensive tool review needed; agents cannot guess and must systematically verify no tool exists

**31 Capability Gap Categories Tested:**

- Physical World Interaction (28 queries, 31%): Smart home, device control, IoT
- Real-Time Sensory Processing (22 queries, 24%): Audio, video, voice commands
- Others: Psychological counseling, parenting advice, financial investment decisions, etc.
## Example records
| User Query                       | Convert this image to JPEG format with 90% quality.                                                                                                                            |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Valid Tools (Equal Function Set) | - Tool 1: convert_to_jpeg (Convert images to JPEG format with specified quality)<br>- Tool 2: image_format_converter (Convert images between different formats including JPEG) |
| Ground Truth                     | Either tool from the equal function set is accepted as correct.                                                                                                                |
| Evaluation Metric                | - EM against equal function set<br>- precision/recall over the valid tool set                                                                                                  |

## 3.2. Metrics

### 3.2.1. Level 1, 2, 5: Exact Match (EM)

- **Definition:** Binary success metric; task succeeds only if agent selects correct tool (L1/L2) or correctly rejects (L5)
- **Calculation:** EM = (# correct) / (# total tasks) × 100%
- **L1 Example:** If 69 of 100 queries correctly identify the exact tool, EM = 69%
- **L5 Example:** If 75 of 91 queries output exact tuple {server: "no", tool: "no"}, EM = 75%
### 3.2.2. Level 3 & 4: Node Set F1 Score

- **Definition:** Evaluates precision and recall of tool selection across multi-step plans
- **Calculation:** F1 = 2 × (Precision × Recall) / (Precision + Recall)
    - **Precision:** (# correctly selected tools) / (# total tools selected)
    - **Recall:** (# correctly selected tools) / (# tools in ground truth)
- **L4 Example:** Task requires [deploy_app (Server A) → scan (Server B) → notify (Server C)]. Agent selects [deploy_app → scan → notify] with correct servers. F1 = 1.0
- **Partial Credit:** Unlike binary EM, F1 rewards partial correctness (e.g., 2/3 tools selected correctly)

### 3.2.3. Efficiency Metric: Normalized Latency (N-Lat.)

- **Definition:** Real wall-clock time relative to MCP-Zero Meta-Llama-3-8B baseline
- **Purpose:** Captures accuracy-efficiency trade-off across architectures
- **Example:** ToolShed-Qwen achieves 5-15× N-Lat. (slower but more accurate); MCP-Zero achieves 1-3× N-Lat. (faster but less accurate)
- **Calculation:** N-Lat. = (Measured Latency) / (Baseline MCP-Zero Latency)

---

# 4. Uniqueness (Why)
## 4.1. Other benchmarks
- [[Seal-Tools]]
- [[NESTful]]
## 4.2. Uniqueness

1. **Architectural Mismatch**: Most large-scale benchmarks (Seal-Tools, NESTful, ToolHop, BFCL v2) model tools as a vast, unstructured flat namespace. This entirely misses the hierarchical, multi-server structure central to the MCP paradigm.
2. **Functional Overlap & Evaluation Reproducibility**: Real-world tool collections inevitably have multiple tools accomplish the same goal. Existing benchmarks handle this poorly:
	- ToolHop meticulously avoids overlap, limiting real-world applicability
	- Others use LLM-as-a-judge, which is costly, biased, and non-reproducible
3. **Fragmented End-to-End Pipeline**: Modern orchestrators comprise retriever + LLM reasoner, but benchmarks evaluate components in isolation:
	- ToolRet: Retriever-only evaluation
	- NestTools, Seal-Tools: Fixed "golden" retriever; only LLM reasoning tested
---

# 5. Generation Method (How)

ETOM's construction proceeds through four systematic stages, each with rigorous quality gates.
## 5.1. Stage 1: Corpus Construction (491 Servers, 2,375 Tools)

- **Data Source:** glama.ai MCP server registry (top 50 servers per category)
- **Filtering Criteria:** Multi-stage process excludes unsuitable servers:
	1. **Trivial Tools:** Functions subsumed by native LLM capabilities (e.g., simple calculators) → excluded
	2. **Meta-Tools:** Designed to augment agent internals (e.g., memory, reasoning patterns) → excluded
	3. **Template Servers:** Developer examples without cohesive use case → excluded
- **Definition Applied:** "Native LLM capability" = tasks a sandboxed LLM could perform without external tools
- **Result:** 491 unique servers with meaningful, externally-focused tools suitable for complex orchestration
## 5.2. Stage 2: Identifying Functional Overlap (Equal Function Sets)

**Challenge:** Multiple tools achieve same outcome; must establish ground truth for evaluation
=> Round-Trip Consistency Approach
- **Phase 2A: Bottom-Up Candidate Generation**
	- For each tool, retrieve top-10 semantically similar tools (Qwen3-Embedding, similarity > 0.8)
	- LLM performs pairwise verification of functional equivalence
	- Union-Find clustering groups verified pairs into connected components
	- **Result:** 145 initial candidate sets

- **Phase 2B: Top-Down Query-Guided Verification**
	- For each L2 query, retrieve top-10 relevant tools via RAG
	- LLM identifies all tools capable of fulfilling the query
	- Cross-check against bottom-up sets for consistency
	- Human validators examine borderline cases (platform-specific nuances)
	- **Result:** 95 high-confidence equal function sets

**Integration:** EFS foundation for L2 evaluation + cornerstone for L4 cross-server compositional tasks

## 5.3. Stage 3: Five-Level Task Generation Curriculum

### 5.3.1. Level 1: Foundational Single-Tool Tasks (781 tasks)
**Pipeline:**
1. **Generation:** Meta-Llama-3-8B generates 2 direct queries per tool, mentioning platform explicitly
2. **Verification:** Second LLM validates intent match, platform mention, unambiguity
3. **Output:** Clear, unambiguous queries with single ground-truth tool
### 5.3.2. Level 2: Context-Aware Tool Retrieval (773 tasks)
**Pipeline:**
1. **Platform Coupling Analysis:** Classify tools as tightly_coupled (platform-iconic) or generic_concept
2. **Dynamic Rule Generation:**
    - For tightly_coupled tools: "You MUST mention the platform {name} because the function is iconic to it"
    - For Generic_concept tools: "You MUST NOT mention the platform; function is generic"
3. **Query Generation:** Rule-based LLM generation with platform mention rules enforced
4. **Verification:** Verifier LLM checks adherence to coupling rules
5. **Round-Trip Consistency Validation:** Validate via RAG retrieval + LLM selection + human cross-check (Lie stage 2)
### 5.3.3. Level 3: Intra-Server Sequential Chaining (327 tasks)
**Pipeline:**
1. **Pseudo Output Schema Inference:** Meta-Llama infers plausible JSON output schema for each tool
2. **Graph Construction:** Build directed graph (nodes=tools, edges=valid input-output flows)
3. **Chain Identification:** Extract simple paths as valid tool chains
4. **Two-Stage Query Generation:**
    - Logical verification: Does chain represent realistic workflow?
    - Query & CoT generation: LLM produces natural query + step-by-step plan
5. **Output:** Multi-step tasks with explicit data flow dependencies

### 5.3.4. Level 4: Cross-Server Compositional Chaining (103 tasks)
**Pipeline:**
1. **Server Sampling:** Group 491 servers into 20+ functional categories; randomly sample 2-4 categories
2. **Feasibility Check:** Qwen/Qwen3-4B evaluates if sampled servers form logical workflow
3. **Workflow Generation:** GPT-4.1 generates cross-server task with explicit dependencies
4. **Quality Control:** Multi-dimensional evaluation (parameter completeness (whether query contains all necessary params) ≥ 7.0, naturalness (whether query maintains conversational style) ≥ 7.0); up to 3 retries
5. **Human Verification:** Manual one-by-one validation for logical coherence
6. **Equal Function Set Integration:** Map ground-truth tools to EFS; verify all feasible alternatives included using LLMs

### 5.3.5. Level 5: Robustness via Capability Gap Identification (91 tasks)
**Pipeline:**
1. **Capability Mapping:** HDBSCAN clustering on tool embeddings; label clusters with high-level capabilities using LLMs
2. **Gap Analysis:** Agentic debate framework identifies functional gaps:
    - "Proposer" LLM brainstorms universal user tasks
    - Check solvability with existing capabilities
    - "Red Team" GPT-4.1 attempts to solve each task
    - Confirm gap if both fail
3. **Persona-Based Query Generation:** Generate diverse queries for each gap (e.g., "Business Analyst", "Student")
4. **Final Verification:** Retrieve semantically similar tools; judge agent confirms no tool can solve
5. **Output:** 31 distinct capability gap categories; only definitively out-of-scope queries included

---

# 6. Contributions
- First hierarchical, end-to-end, cross-server tool calling benchmark
- Deterministic metrics besides LLM-as-judge using Equal Function Set as a base

---

# 7. Gaps

- ETOM uses simulated execution via pseudo-output schemas rather than live API calls
	=> Incorporate edge-case simulations (malformed outputs, timeout handling) to further test agent resilience

- At L5, correct rejection largely emerges from backbone model's reasoning rather than from architectural checks
	=>Develop dedicated modules or prompting strategies specifically designed for out-of-scope detection

- Limited Domain Scope
	=> Future Expansion Paths:
	- Multilingual task generation
	- Additional server sources beyond glama.ai
	- Compositional task complexity scaling

- Maximum plan length in L4 is 3.83 steps; real workflows can be much longer
	=> Extended L5-equivalent tasks with 10-20+ step workflows to evaluate memory bottlenecks

- Does not deeply analyze reasoning quality beyond success/failure
	=> Community extensions can leverage released dataset to perform fine-grained error analysis

---
# 8. Highlights
> [!PDF|255, 208, 0] [[ETOM AFive-Level Benchmark for Evaluating Tool Orchestration within MCP Ecosystem.pdf#page=2&annotation=1687R|ETOM AFive-Level Benchmark for Evaluating Tool Orchestration within MCP Ecosystem, p.1454]]
> > ETOM successfully exposes failure modes in orchestration and robustness that are missed by benchmarks with a narrower scope.


> [!PDF|255, 208, 0] [[ETOM AFive-Level Benchmark for Evaluating Tool Orchestration within MCP Ecosystem.pdf#page=2&annotation=1690R|ETOM AFive-Level Benchmark for Evaluating Tool Orchestration within MCP Ecosystem, p.1454]]
> > without co-designed, hierarchy-aware reasoning strategies, such structures can actually introduce new failure modes and degrade performance

