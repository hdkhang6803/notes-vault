---
Tags:
Date: February 10, 2026
Authors: Yiming Xiong, Shengran Hu, Jeff Clune (University of British Columbia / Vector Institute)
Venue:
Paper: "A Systematic Literature Review of Agentic AI: Definitions, Architectures, and Challenges"
---
# 1. Terminology

| **Term**                       | Definition                                                                                                                                                                     |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Agentic AI**                 | AI systems that actively pursue objectives, adapt strategies, and initiate actions in dynamic environments (distinct from purely reactive or tool-based systems)               |
| **Agency**                     | Capacity to act independently with goal-directedness and intentional behavior; implies selecting, constructing, or interpreting objectives (contrasted with mere autonomy)     |
| **Autonomy**                   | Operational independence: ability to function without constant external control, but without necessarily implying goals or long-term planning                                  |
| **BDI Architecture**           | Belief-Desire-Intention model; formalizes agent decision-making via internal representations of beliefs (environment state), desires (goals), and intentions (committed plans) |
| **GOFAI**                      | Good Old-Fashioned AI; symbolic, rule-based systems that are transparent but rigid, lacking adaptivity or generalization                                                       |
| **Intersection Analysis**      | Novel analytical method mapping each paper's primary contribution type against its addressed research gap, surfacing hotspots and deserts in the research landscape            |
| **Adaptability Desert**        | Term coined by this paper for the near-total absence of constructive frameworks, models, or methods addressing dynamic, unforeseen environment adaptation                      |
| **Exploitation Phase**         | Characterization of the current field state: heavily applying and refining existing autonomous capabilities rather than exploring novel architectures                          |
| **Metacognitive Architecture** | Proposed future architecture featuring distinct modules for self-reflection and real-time plan modification, as opposed to static LLM execution chains                         |
| **Resilience Benchmark**       | Proposed evaluation paradigm measuring agent recovery rate from induced environmental errors (e.g., Chaos Engineering), contrasted with static accuracy metrics                |

---
# 2. Categories

Papers are classified along two axes: the five **research fields** of the taxonomy, and the **contribution type** × **research gap** matrix used in the intersection analysis.

**Taxonomy fields:**

|Field|Description|
|---|---|
|Memory & Cognition|Improving reasoning, long-term memory retention, and cognitive architectures for context-aware intelligence|
|Networking & Systems|Distributed frameworks, multi-agent coordination, scalability, and system-level integration|
|Trust & Safety|Security, robustness, ethics, and trustworthiness in adversarial or unpredictable environments|
|Evaluation & Limits|Benchmarking methodologies, capability measurement frameworks, and theoretical boundaries of agency|
|Applications & Use-Cases|Domain-specific implementations demonstrating practical utility (healthcare, agriculture, finance, education, etc.)|

**Contribution types** (Table 4): Survey, Framework, Method, Model, Evaluation, Application, Theory.

**Research gap categories** (Table 5): New Models, Scalability, Autonomy, Generalization, Trust, Efficiency, Human-AI Collaboration.

---

# 3. Review Protocol
![[Agentic-AI.pdf#page=4&rect=33,350,543,715&color=yellow|Agentic-AI, p.36179]]

**Research Questions:**
- MQ: What are the general approaches to Agentic AI research?
	- SQ1: How does the literature define and delimit "Agentic AI"?
	- SQ2: What architectures, frameworks, and design standards predominate?
	- SQ3: Which application domains already demonstrate practical value?
	- SQ4: What risks, evaluation challenges, and research gaps remain open?

**Search Strategy:**
- Databases: ACM Digital Library, Elsevier, IEEE Xplore, Nature, ScienceDirect, Springer Link
- Query string (applied to titles and full texts): `"agentic ai" OR "AI agent" OR "autonomous agent" AND ("LLM" OR "large language model")`
- Period: 2021–mid 2025

**Inclusion/Exclusion Criteria:**

- EC1: Exclude studies not published in scientific journals or conference proceedings
- EC2: Exclude studies not directly related to Agentic AI
- EC3: Exclude duplicates retrieved across databases
- Remaining ambiguous cases resolved by full-text reading; inter-rater (2 people) disagreements resolved by a third senior researcher

**Quality Assessment Criteria (QE1–QE7):** Each retained paper was evaluated for: 
- clear research purpose (QE1)
- adequate background/context (QE2)
- related work coverage (QE3)
- architecture or methodology description (QE4)
- reported results (QE5)
- objective-aligned conclusion (QE6)
- future work recommendations (QE7).
![[Agentic-AI.pdf#page=6&rect=35,519,540,715&color=yellow|Agentic-AI, p.36181]]
---

## 4. Results

**Study Selection (PRISMA-aligned):**

|Stage|Count|
|---|---|
|Initial retrieval|125|
|After impurity removal (EC1, EC2)|115|
|After duplicate removal (EC3)|109|
|After title/abstract screening|79|
|After full-text screening|52|
|After quality assessment|**48**|

**Distribution by database:** ACM (20), Elsevier (12), IEEE (8), Springer (3), ScienceDirect (3), Nature (2).

**Distribution by year:** Near-zero activity 2021–2023; sharp spike in 2024 (n=35); moderate in early 2025 (n=9), reflecting a rapidly emerging but still volatile field.

**Distribution by country:** US dominates (21 papers); India and China tied second (5 each); UK fourth (4); Italy and Australia minor contributors (2 each); remainder single-paper countries.

**Distribution by taxonomy field:**

| Field                    | Share |
| ------------------------ | ----- |
| Applications & Use-Cases | 31.9% |
| Networking & Systems     | 27.7% |
| Evaluation & Limits      | 27.7% |
| Trust & Safety           | 8.5%  |
| Memory & Cognition       | 4.3%  |

---

## 5. Findings

**SQ1 - Definitions:** No consensus definition exists. Two dominant interpretations: 
- (1) Agentic AI as a synonym for autonomous agents with explicitly encoded decision-making (BDI, RL)
- (2) Agentic AI as an emergent property of LLMs embedded in tool-use and feedback loops. 
=>The paper proposes an operational definition around four measurable characteristics: Autonomy, Goal-Directedness, Adaptivity, and Proactivity. Few deployed systems satisfy all four simultaneously.

**SQ2 - Architectures:** Three lineages are identified:
- Classical: symbolic GOFAI and BDI agents (transparent but brittle)
- Modern pre-LLM: MAS and RL agents (adaptive and goal-directed but lacking explicit belief/intention representations)
- LLM-based: AutoGPT, ReAct, BabyAGI-style systems exhibiting emergent agency through tool use, task decomposition, and recursive prompting, but without formal cognitive architectures. 
=> No dominant architectural standard has emerged; LLM-based approaches are ascendant but poorly standardized.

**SQ3 - Application domains:** 
- Healthcare (Alzheimer's prediction, oncology, clinical calculations, rheumatology)
- industrial automation (robot control, manufacturing, supply chains)
- agriculture (disease detection, edge-based crop classification)
- finance (risk modeling, GDPR compliance automation)
- education (active learning agents)
- security (SOC automation, forensics dataset generation).

**SQ4 - Risks, evaluation gaps, and research gaps:**

_Evaluation:_ 
- No standard benchmarks for agency exist. 
- Traditional metrics (accuracy, reward) fail to capture proactivity, long-term goal pursuit, and adaptation to novel environments. 
- Stochasticity and opacity in LLM-based agents complicate reproducibility.

_Technical risks:_ 
- Generalization failure outside narrow contexts
- cascading fragility from tight module coupling
- resource intensity of foundation-model-based agents
- prompt management and context tracking overhead.

_Ethical/societal risks:_ 
- Accountability gaps in autonomous decision-making
- anthropomorphism and trust miscalibration
- bias amplification (e.g., ageist attitudes in LLMs reinforcing human ageism)
- vulnerability risks from automated SOCs.

_Intersection analysis — key finding:_ The Autonomy gap is the only one addressed across nearly all contribution types, confirming the field is in an exploitation phase. Critically, Adaptability has zero constructive contributions (no frameworks, models, methods, or applications): only one survey and one evaluation exist on the topic. The New Models gap is similarly barren, with no frameworks or methods proposing fundamentally new architectures.

# 8. Highlights

> [!PDF|255, 208, 0] [[Agentic-AI.pdf#page=1&annotation=768R|Agentic-AI, p.36176]]
> >  evolving state of Agentic AI, analyzing peer-reviewed studies published between 2021 and 2025

> [!PDF|255, 208, 0] [[Agentic-AI.pdf#page=1&annotation=771R|Agentic-AI, p.36176]]
> >  five distinct research fields: Memory Cognition, Networking Systems, Trust Safety, Evaluation Limits, and Applications Use-Cases.

> [!PDF|255, 208, 0] [[Agentic-AI.pdf#page=1&annotation=774R|Agentic-AI, p.36176]]
> > understood as the capacity of an entity to act autonomously and intentionally.

> [!PDF|255, 208, 0] [[Agentic-AI.pdf#page=2&annotation=777R|Agentic-AI, p.36177]]
> >  Kitchenham’s methodology 

> [!PDF|255, 208, 0] [[Agentic-AI.pdf#page=2&annotation=783R|Agentic-AI, p.36177]]
> > novel Intersection Analysis that maps research contributions against identified gaps.

