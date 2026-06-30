---
Tags:
Date: "2026"
Authors: M Gridach, J Nanavati, K Zine El Abidine, C Yacoubian, C Mack
Venue: ACM Computing Surveys
Paper: "Agentic AI in Healthcare: Opportunities, Challenges, and Future Directions"
---
# 1. Terminology

| Term                                 | Definition                                                                                                     |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| Agentic AI                           | AI systems capable of autonomous perception, reasoning, acting, and learning, also called compound AI systems  |
| Single-Agent System                  | An agent that achieves goals independently without relying on other AI agents, possibly with human-in-the-loop |
| Multi-Agent System (MAS)             | A composed system of two or more interacting agents, each specializing in a subtask or domain                  |
| LM-based Agent                       | A single agent with an LLM backbone capable of reasoning, planning, and tool execution                         |
| Autonomy Levels (L1–L5)              | A 5-level taxonomy: L1 Operator, L2 Collaborator, L3 Consultant, L4 Approver, L5 Observer                      |
| Retrieval-Augmented Generation (RAG) | Technique combining retrieval from external knowledge sources with LLM generation                              |
| EHR (Electronic Health Record)       | Digital patient records used as primary data sources for clinical agents                                       |
| Hallucination                        | LLM output that is nonsensical or unfaithful to source content; a major safety risk in clinical settings       |
| Prompt Injection                     | Attack where adversarial instructions in inputs override an agent's intended behavior                          |
| Blackboard Architecture              | Historical multi-agent paradigm using a shared workspace for heterogeneous knowledge sources to collaborate    |
| Confidence-Based Escalation          | Mechanism where agent actions below a confidence threshold are flagged for human review instead of executed    |
| Runtime Governance                   | Structured mechanisms to monitor, constrain, and audit agent behavior during execution                         |
| MDT (Multidisciplinary Team)         | Collaboration structure across medical specialties; mirrored in multi-agent frameworks like RareAgents         |
| SaMD                                 | Software as a Medical Device: regulatory classification relevant to AI health tools                            |
| Tool Calling Efficiency              | Metric evaluating how accurately and effectively agents utilize external tools                                 |
| Interoperability                     | The ability of different systems, devices, or applications to connect, communicate, and share data seamlessly  |

---

# 2. Review Protocol
<span style="color:rgb(192, 0, 0)">No explicit pipeline</span>
- **Coverage period:** 2020–mid-2025 (finalized February 2025)
- **Source types:** Peer-reviewed papers, preprints, and key open-source frameworks
- **Inclusion criteria:** Works introducing agentic behaviors (autonomy, memory persistence, multi-agent collaboration, tool integration); foundational systems included for historical context
- **Selection basis:** Relevance, novelty, and contribution to the taxonomy and themes of the survey
- **Scope:** Dedicated focus on Agentic AI in healthcare; excludes general-domain or non-medical agent surveys

---

# 3. Categories

The paper organizes healthcare Agentic AI across several dimensions:

**By agent architecture:**

- Single-agent systems (e.g., EHRAgent, LLM-MedQA)
- Multi-agent systems (e.g., MDAgents, MedAgents, TriageAgent, RareAgents)

**By clinical application domain:**

- EHR interaction (EHRAgent, EHRFlow, MedAssist)
- Clinical triage (TriageAgent)
- Medical question answering (LLM-MedQA, MedAgents, AgentClinic)
- Disease diagnosis (MDAgents, RareAgents, PathFinder, MEDDxAgent, Zodiac)
- Mental health (AutoCBT, PsyDraw)
- Reasoning & explainability (ArgMed-Agents, MedAide, PRefLexOR)
- Scientific discovery (ProtAgents, SciAgents)
- Outpatient / simulation (PIORS, Agent Hospital)

**By challenge domain:**

- Security & privacy
	- Adversarial attacks
	- Hallucinations
	- Malicious, fake content
	- Poisoning: model, data, RAG, agent
	- Lack runtime governance
- Interoperability: interactions between agents in MAS
	- REST API (JSON), protobuf
	- Agent communication language
	- Natural language
- Scalability & resource management
	- Horizontal scaling
	- Vertical scaling
- Human oversight formalization
	- Structure control point: pre, mid, post execution
	- confidence-based escalation
	- Decision audit trails
- Ethics, regulation, and governance

**By benchmark type:**

- Medical QA (MedQA, PubMedQA, JAMA, Medbullets)
- Diagnostic reasoning (DDXPlus, SymCat)
- Medical visual interpretation (PMC-VQA, PathVQA, MedVidQA, MIMIC-CXR)

**By metrics:**
- Accuracy metrics: Score and success rate
- Scalability (Stress testing)
- Tool-calling efficiency: correctness, correct final state, intermediate step correctness, ratios of steps taken to expected steps
- Security
- Privacy: differential privacy, statistical evaluation of identifiability, privacy leakage quantification
- Task adherence: task completion success rates adherence to task priorities, deviation from predefined objectives, and recovery time from distractions or interruptions
- Self-behavior modeling and updating: adaptation success rate, model accuracy over time, resource eficiency, and convergence speed
- Evaluating chatbots: Abbasian et al. 

---

# 4. Findings

- Most healthcare Agentic AI work emerged only recently, with significant developments beginning in **2024**; the field is heavily dominated by conversational agents in earlier work
- **Multi-agent architectures outperform single-agent baselines** on medical reasoning tasks across multiple benchmarks (MedQA, MedMCQA, PubMedQA), but **more agents does not always improve performance** — MDAgents found N=3 with adaptive assignment optimal
- **No surveyed system has achieved clinical deployment** with regulatory approval or prospective validation; PsyDraw and PIORS are the closest approximations (pilot settings only)
- **Safety mechanisms are largely implicit** (prompting, role-based constraints, post-hoc eval); only TriageAgent and EHRFlow use explicit confidence-based gating; none provide provable safety bounds
- **Scalability analysis is absent** from most frameworks — latency, cost-per-query, and concurrent workload behavior are rarely characterized
- ToolEmu shows **69% of emulated failures manifest with real tools**
- AgentClinic reports **order-of-magnitude diagnostic accuracy drops** in interactive, multi-modal settings vs. static QA, revealing long-horizon brittleness
- **Hallucination and bias** are documented in Med-HALT and clinical studies (Med-PaLM 2), with non-uniform safety across populations
- **Regulatory readiness varies globall**y: the **EU AI Act** takes the most proactive stance (Agentic AI in healthcare likely "high-risk"); the US FDA has no agent-specific rules yet
- **Computational costs for multi-agent healthcare systems are significant:** MDAgents estimated ~$172,705 to run GPT-4 (Vision) on full test sets; EHRAgent up to $0.60/query on MIMIC-III
- **Standard LLM evaluation metrics fail to capture** trust-building, ethical compliance, user comprehension, and emotional support — dimensions critical for patient-facing agents
- OMEGA benchmark reveals **key LLM reasoning failure modes**: overthinking/recursive error spirals, heuristic guessing, and limited compositional reasoning
- **Benchmark-reality gap** — most benchmarks (MedQA, DDXPlus, etc.) lack fidelity to real clinical records, multimodal inputs, and sequential workflow dynamics present in actual care settings. 
- **Calibration–autonomy mismatch** — LLM agents exhibit systematic overconfidence (documented by R-Judge), making confidence-based escalation mechanisms unreliable in practice. 
- **Compounding error in multi-step workflows** — no existing framework formally models or mitigates the propagation of errors across sequential clinical decisions (history-taking → diagnosis → treatment). 
- **Multimodal data fusion** — modality heterogeneity, temporal misalignment, and modality imbalance remain under-addressed; annotated multimodal clinical datasets are scarce and costly to curate. 
- **Accountability gaps in MAS** — responsibility attribution in collaborative multi-agent settings is ambiguous when one agent fails due to adversarial tampering or misalignment. 
- **Runtime governance is absent** — no lightweight deployment-time mechanism exists to enforce safety guarantees without introducing prohibitive latency in time-critical environments.

---

# 6. Future Directions

- Develop **deployment-oriented design** incorporating formal safety specifications, scalability engineering, and regulatory pathway planning from the outset — moving beyond benchmark-driven evaluation
- Build **error-aware planning mechanisms** that quantify cumulative uncertainty across reasoning chains and trigger escalation before downstream decisions are contaminated
- Design **calibration methods specific to agentic reasoning chains** (not just single-turn outputs), and escalation mechanisms robust to miscalibration (e.g., uncertainty ensembles, independent verification agents)
- Develop **lightweight runtime governance** components: action validation, real-time audit trails, confidence thresholding, and policy engines encoding HIPAA/GDPR constraints as enforceable rules
- Advance **multimodal fusion architectures** capable of integrating unstructured text, structured EHR, medical images, and video in unified agentic reasoning pipelines
- Design **cost-effective, low-latency Agentic AI architectures** tailored to clinical constraints — including hybrid small/large model approaches and decentralized architectures
- Incorporate **user-centered evaluation metrics** (trust, empathy, ethical compliance, comprehension) alongside task-performance metrics for patient-facing systems
- Formalize **human oversight** through structured control points (pre-execution gates, mid-execution checkpoints, post-execution review) with domain-calibrated confidence thresholds
- Pursue **federated learning** approaches to train robust agents across hospital networks while preserving privacy — particularly relevant for multisite clinical research
- Ensure **equitable access** to Agentic AI across health systems to avoid widening the gap between patients who can and cannot benefit from this technology

---

# 7. Memory/Context Related

**Memory in agent frameworks (from Table 1):**

|Framework|Memory Mechanism|
|---|---|
|AutoGen|Short-term context chaining; extendable with external memory|
|LangGraph|Integrates LangChain memory modules (buffer, summary, retriever)|
|AutoGPT|Memory via vector stores|
|CrewAI|Persistent memory via shared context objects|
|MetaGPT|Memory modules manage planning and document history|
|Letta|Long-running agents with persistent state; recall and memory logs|
|CoALA|Abstract memory and planning modules modeled after human cognitive architecture|
|BabyAGI|Vector database for persistent task memory|
|LlamaIndex|Memory-like retrieval through persistent knowledge graphs and embeddings|
|MemGPT|Emphasizes memory and reasoning capabilities for long-term agent autonomy|

**Memory in healthcare agents:**

- **RareAgents** explicitly uses **long-term memory retrieval** as a core component of its MDT coordination pipeline for rare disease diagnosis
- **Memory persistence** is listed as a standard evaluation metric for agents (Table 6) — measuring how well an agent retains relevant information over time
- **Context window limitations** are flagged as a scalability bottleneck: LLM-based agents are hindered from managing complex, multi-step tasks or long-term collaboration due to finite context windows

**Memory-related limitations:**

- No healthcare agent framework provides **episodic or semantic memory** with formal grounding — memory is either vector-store-based retrieval or short-term context chaining
- **Long-horizon brittleness** is a key failure mode: agent reliability degrades sharply as task length increases (documented by METR's autonomous task time-horizon analysis), which directly implicates the limits of working memory and context management in clinical pathways
- **Compounding errors in multi-step workflows** are partly a memory failure — agents lose track of prior reasoning state across sequential clinical decisions, with no framework formally modeling this propagation

**Relevant benchmarks for memory/context:**

- **AgentClinic** — evaluates agents in interactive, multi-turn clinical simulations; shows memory/context degradation relative to static QA
- **Agent Hospital** — multi-agent simulator where memory across patient/doctor/nurse turns is implicitly tested
- **MedChain-Agent** — specifically designed for interactive sequential benchmarking with simulated patient/EHR environments, testing multi-turn context handling

---

# 8. Highlights
> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=7&annotation=700R|Agentic AI in healthcare, p.7]]
> >  none of the surveyed systems provide provable safety bounds or certiied constraint enforcement at runtime.


> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=7&annotation=703R|Agentic AI in healthcare, p.7]]
> > most frameworks have been evaluated on small-scale benchmarks with ixed agent


> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=8&annotation=709R|Agentic AI in healthcare, p.8]]
> > ystematic scalability analysis including latency proiling, cost-per-query estimates, and behavior under concurrent workloads remains absent from the literature.

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=8&annotation=712R|Agentic AI in healthcare, p.8]]
> > ll surveyed systems operate in research or prototype settings, with no framework reporting regulatory approval, prospective clinical validation, or integration with production EHR systems. 

Security&Privacy
> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=11&annotation=748R|Agentic AI in healthcare, p.11]]
> > ack of structured runtime governance mechanisms that actively monitor, constrain, and audit agent behavior during execution.

Interoperability
> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=13&annotation=763R|Agentic AI in healthcare, p.13]]
> > Additional protocols must address challenges of scalability, coordination, and secure communication

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=13&annotation=766R|Agentic AI in healthcare, p.13]]
> > Enhanced standards for multi-modal communication could improve agent interactions using diferent sensory inputs, such as visual and auditory data.

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=13&annotation=769R|Agentic AI in healthcare, p.13]]
> > d protocols that promote trust, enforce privacy and conidentiality requirements, and establish accountability in agent interactions.

Scalability & Resource management 
> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=13&annotation=784R|Agentic AI in healthcare, p.13]]
> > these approaches face challenges like increased communication overhead and resource allocation

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=13&annotation=787R|Agentic AI in healthcare, p.13]]
> > High resource demands restrict real-time responsiveness and the ability to deploy large numbers of agents simultaneously.

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=13&annotation=790R|Agentic AI in healthcare, p.13]]
> > Addressing these challenges involves hybrid approaches that integrate smaller and task-speciic models with LLMs, which are gaining traction

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=13&annotation=793R|Agentic AI in healthcare, p.13]]
> > ecentralized architectures, where agents share LLM outputs or pre-processed embeddings, reduce redundant queries

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=14&annotation=805R|Agentic AI in healthcare, p.14]]
> >  the costs associated with scaling Agentic AI across health systems are signiicant.

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=14&annotation=811R|Agentic AI in healthcare, p.14]]
> > Federated learning supports local model training on edge devices

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=14&annotation=814R|Agentic AI in healthcare, p.14]]
> >  prioritizing tasks based on energy requirements and oloading non-critical tasks, f

> [!PDF|8, 109, 221] [[Agentic AI in healthcare.pdf#page=14&annotation=820R|Agentic AI in healthcare, p.14]]
> >  Adaptive model compression

Human oversight
> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=15&annotation=850R|Agentic AI in healthcare, p.15]]
> > onidence calibration remains an open challenge for LLM-based agents [81 , 101 ], and uncalibrated scores may lead to either excessive escalation or dangerous under-escalation.


Ethical,

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=15&annotation=853R|Agentic AI in healthcare, p.15]]
> > patients interacting with mental health chatbots or virtual therapists may develop emotional reliance on agents, avoiding qualiied professionals and weakening essential human connections in care

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=15&annotation=856R|Agentic AI in healthcare, p.15]]
> > Addictiveness and dependency, over-reliance on AI agents can lead to compulsive use, 

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=15&annotation=859R|Agentic AI in healthcare, p.15]]
> > f one agent behaves unethically due to adversarial tampering or incomplete alignment, responsibility attribution becomes ambiguous, and the integrity of the entire system may be compromised. 

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=16&annotation=862R|Agentic AI in healthcare, p.16]]
> > autonomous agents process sensitive data with limited human oversight. A

 Regulatory, Practical considerations
 > [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=16&annotation=877R|Agentic AI in healthcare, p.16]]
> >  FDA is focusing on creating new guidance for the use of AI in pharmaceutical research and AI working groups to streamline policy development
> 
> 


Metrics
> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=18&annotation=918R|Agentic AI in healthcare, p.18]]
> >  signiicant gaps remain in maintaining task focus on settings with competing objectives or noisy signals, and over long-term tasks.


> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=19&annotation=912R|Agentic AI in healthcare, p.19]]
> > hallenges persist in implementing eicient, real-time self-updating mechanisms without compromising performance. 

> [!PDF|255, 208, 0] [[Agentic AI in healthcare.pdf#page=19&annotation=915R|Agentic AI in healthcare, p.19]]
> >  address these challenges by developing lightweight and decentralized self-updating mechanisms that can seamlessly integrate into multi-agent frameworks

> [!PDF|234, 82, 82] [[Agentic AI in healthcare.pdf#page=19&annotation=924R|Agentic AI in healthcare, p.19]]
> > valuation metrics neglect pivotal aspects such as trust-building, ethical compliance, user comprehension, and emotional suppor

> [!PDF|8, 109, 221] [[Agentic AI in healthcare.pdf#page=19&annotation=932R|Agentic AI in healthcare, p.19]]
> >  AI agents incorporate these user-centered metrics alongside the task-performance and security metrics

 > [!PDF|234, 82, 82] [[Agentic AI in healthcare.pdf#page=19&annotation=932R|Agentic AI in healthcare, p.19]]
> >  AI agents incorporate these user-centered metrics alongside the task-performance and security metrics


