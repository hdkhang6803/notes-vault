---
Tags:
Date: February 10, 2026
Authors: Yiming Xiong, Shengran Hu, Jeff Clune (University of British Columbia / Vector Institute)
Venue:
Paper: "Function Calling in Large Language Models: Industrial Practices, Challenges, and Future Directions"
---
# 1. Terminology

| Term                   | Definition                                                                                                                                |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Function Calling       | LLMs' ability to interpret user requests and invoke predefined computational procedures to deliver precise, contextually suitable answers |
| Function               | A predefined block of code, software, or tool that accepts parameters and returns a result                                                |
| Pre-call Stage         | Pipeline phase covering query processing and function retrieval before any invocation                                                     |
| On-call Stage          | Pipeline phase covering function triggering ("when to call") and parameter extraction ("how to call")                                     |
| Post-call Stage        | Pipeline phase covering function execution and natural language response generation                                                       |
| Function Hallucination | When an LLM invokes non-existent or inapplicable functions, or populates parameters that have no legitimate existence                     |
| SFT                    | Supervised Fine-Tuning: direct imitation learning on curated input-output pairs                                                           |
| PEFT                   | Parameter-Efficient Fine-Tuning  e.g., LoRA; fine-tunes a small subset of parameters to preserve general capabilities                     |
| RLHF                   | Reinforcement Learning from Human Feedback: aligns model behavior with complex human preferences                                          |
| RAG                    | Retrieval-Augmented Generation: augments LLM outputs by retrieving relevant past examples or documents                                    |
| AST                    | Abstract Syntax Tree: used for offline evaluation of function call structure without live API access                                      |
| BFCL                   | Berkeley Function Calling Leaderboard: a standard benchmark for evaluating function calling performance                                   |
| Seesaw Effect          | The tradeoff where fine-tuning for function calling degrades general language capabilities, particularly QA                               |

---

# 2. Categories

The paper organizes the literature along the three stages of the function calling pipeline, then by solution type:
![[Function-calling.pdf#page=3&rect=43,357,443,647&color=yellow|Function-calling, p.3]]

| Category                                           | Sub-category                                                                                                                                                                                                               | Representative Works                            |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| **Pipeline Challenges**                            | **Pre-call:** <br><br>Intent Recognition (Query processing)<br><br>Function Redundancy (Function Retrieval)                                                                                                                | GeckOpt, Gorilla, COLT, Confucius               |
|                                                    | **On-call:** <br><br>Missing/Unnecessary Calls **(When to call)**<br><br>Missing/illegal Parameters, Function Hallucination, Pronoun Resolving, LLM's latency & accuracy, Multi-call, Context Management **(How to call)** | Hammer, ChemAgent                               |
|                                                    | **Post-call:** Result Mismatch, Irrelevant Output, Semantic-Code Gap, Execution Failure                                                                                                                                    | —                                               |
| **Sample Construction & Fine-tuning**              | Function collection (manual, LLM-generated, web-mined)                                                                                                                                                                     | GPT-4, Qwen, LLaMA                              |
|                                                    | Sample construction (text/token representation, multi-turn)                                                                                                                                                                | Toolformer, ToolGen, GraphQL-RestBench, Hammer  |
|                                                    | Fine-tuning (SFT, PEFT, RL, RLHF)                                                                                                                                                                                          | GPT4Tools, CITI, WebGPT, FunRL, ToolRL, DPO     |
| **Deployment & Inference (Mitigation Strategies)** | Task planning (foundational, GUI, LLM optimization, Error handling, tree-based decision making, adaptive)                                                                                                                  | ReAct, LLM-MCTS, ControlLLM, AppAgent, OS-ATLAS |
|                                                    | Context engineering (text, multimodal)                                                                                                                                                                                     | NexusRaven, FunReason, MLLM-Tool, VisionGPT     |
|                                                    | Function generation & selection                                                                                                                                                                                            | CFG, TOOL-ED, PAE                               |
|                                                    | Function mapping (pronoun, format, error checking, privacy)                                                                                                                                                                | Syllabus, PrivacyChecker, Gatekeeper            |
|                                                    | Response generation (templates, review, RAG)                                                                                                                                                                               | ToolLLM, NexusRaven, SFR-RAG                    |
|                                                    | Memory schemes                                                                                                                                                                                                             | MemoryBank, SCM, TiM, RET-LLM                   |
|                                                    | Latency optimization                                                                                                                                                                                                       | vLLM, FlashAttention, LLM Compiler, TinyAgent   |
| **Evaluation**                                     | Metrics + benchmarks                                                                                                                                                                                                       | API-BLEND, ToolBench, RotBench, ToolEyes        |
| **Open Issues / Future**                           | 1. Service quality, Function Usability, Feedback RL<br><br>2. Function isolation, Post processing strategy<br><br>3. LLM–industrial system bridgin<br><br>4. standardization and design for integration                    | —                                               |

![[Function-calling.pdf#page=11&rect=50,191,446,324&color=yellow|Function-calling, p.11]]
![[Function-calling.pdf#page=18&rect=47,185,451,460&color=yellow|Function-calling, p.18]]

---

# 3. Review Protocol

This paper is a **structured expert survey**, not a strict SLR. It does not declare formal research questions, a reproducible search strategy, or inclusion/exclusion criteria. The coverage is driven by the authors' taxonomy of the function calling pipeline.

- **Research Questions:** Not explicitly stated; the implicit driving question is: _"What are the challenges, solutions, and future directions for function calling in LLMs from an industrial perspective?"_
- **Search Strategy:** Not reported (no databases, query strings, or date ranges disclosed)
- **Inclusion/Exclusion Criteria:** Not reported
- **Quality Assessment Criteria:** Not reported

---

# 4. Results

- **Study Selection:** No PRISMA flowchart; paper count not reported
- **Data Extraction:** Organized via a manually constructed taxonomy across 6 dimensions (definition/workflow, industrial challenges, sample construction, deployment & inference, evaluation, futures)
- **Quality Scores:** Not applied; coverage is assessed qualitatively via Table 1, which compares 14 prior surveys across the 6 dimensions — this paper is the only one to cover all six

---

# 5. Findings

**By pipeline stage and challenge:**

**Pre-call.** Intent recognition (C1.1) remains heavily reliant on LLM base capabilities and prompt design, with limited personalization. Function redundancy (C1.2) in large tool pools is addressed via sparse retrieval (BM25, Gorilla), dense retrieval (Confucius/SentenceBERT), and graph-based retrieval (COLT/GNN).

**On-call.** Missing calls (C2.1) and unnecessary calls (C2.2) are mitigated through fine-tuning with diverse samples including no-call scenarios, and techniques like function masking (Hammer). Parameter extraction faces three compounding issues: missing/illegal parameters (C3.1), hallucination (C3.2), and pronoun/temporal reference resolution (C3.3) — addressed via ToolGen (token-level tool encoding), LoRA-based PEFT, and mapping modules. Multi-call orchestration (C3.5) and context management across turns (C3.6) are handled by planning frameworks (ReAct, LLM-MCTS) and memory schemes (MemoryBank, SCM).

**Post-call.** The semantic-to-code gap (C4.3) is the central post-call challenge: LLM outputs in natural language must be mapped to executable system calls via pronoun resolution, strict format alignment, and parameter validation. Execution failures (C4.4) and result mismatches (C4.1) are addressed through review mechanisms, agent correction loops, and RLHF.

**Training insights (empirical).** Two key findings from the authors' own experiments stand out as research-fertile: (1) function calling ability saturates quickly (~400 high-quality samples), making _data quality more important than quantity_; (2) a **seesaw effect** exists — FC fine-tuning substantially improves AST accuracy but degrades QA performance, suggesting that mixed training with natural language data is necessary to preserve general capabilities. Additionally, base models below 7B parameters show near-zero function calling ability even after training, indicating a hard capability threshold.

**Open issues.** Four industrial gaps are identified: unified evaluation of latency + accuracy + security; function isolation and post-processing for regulatory compliance; bridging LLM agents with legacy deterministic pipelines; and the absence of standardized function design conventions for LLM integration.

# 8. Highlights