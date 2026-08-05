---
Tags:
Date: "2024"
Authors: Jiaxuan You1, Mingjie Liu1, Shrimai Prabhumoye1, Mostofa Patwary1, Mohammad Shoeybi1, Bryan Catanzaro
Venue: EMNLP
Paper: "LLM-Evolve: Evaluation for LLM’s Evolving Capability on Benchmarks"
---
## 1. Terminology

| Term                       | Definition                                                                                                                                                                                                                                                                                     |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| LLM-Evolve                 | An evaluation framework that converts a standard LLM benchmark into a sequential, multi-round problem-solving setting by reusing an LLM's own past inputs, outputs, and feedback as few-shot demonstrations in later rounds, without altering the benchmark's original test set or metric.     |
| Demonstration memory (𝒟)  | A growing store of tuples $𝒟 = {(x^{lm}_i, y^{lm}_i, f_i)}$, where $x^{lm}_i$ and $y^{lm}_i$ are an LLM's own input and generated output from a prior round and f_i ∈ {True, False} is binary feedback on whether that output was correct/desirable.                                          |
| Feedback (f_i)             | A binary label attached to each stored experience indicating correctness; sourced either from the benchmark's ground-truth answer key or, in the ablation, from a separate LLM (e.g., Llama3-70B) judging the output.                                                                          |
| Retriever (r_φ)            | A dense embedding function (Contriever or BERT) used to encode both the new query x and stored past inputs $x^{lm}_j$ into vectors, enabling nearest-neighbor selection of relevant past experiences.                                                                                          |
| Top-k retrieval            | The selection rule i ∈ topk-min over $x^{lm}_j$ ∈ 𝒟 with $f^{lm}_j$ = True, choosing the k stored positive experiences whose embeddings $r_φ(x^{lm}_j)$ are closest in L2 distance to $r_φ(x)$, to replace the benchmark's originally fixed few-shot demonstrations $\{x^{demo}, y^{demo}\}$. |
| Multi-round LLM-Evolve     | The iterative procedure in which, after each round produces new outputs $y^{LLM-Evolve}$ via Eq. 3, the demonstration memory 𝒟 is refreshed with these new experiences and used to initiate the next round of evaluation.                                                                     |
| Multi-turn extension       | A generalization of 𝒟 for interactive, multi-turn benchmarks (e.g., AgentBench), storing full interaction trajectories $𝒟 = {(x^{lm}_{i1}, y^{lm}_{i1}, ..., x^{lm}_{it}, y^{lm}_{it}, f_i)}$ and retrieving based on similarity of the first turn's input $x^{lm}_{i1}$.                    |
| Standard setting (round 0) | The conventional i.i.d. benchmark evaluation, $y^{lm} = p_θ(x, {x^{demo}_i, y^{demo}_i})$, using the benchmark's original fixed few-shot demonstrations.                                                                                                                                       |
| LLM-Evolve Gain            | The reported performance delta between the best LLM-Evolve round and the Standard (round 0) accuracy for a given model on a given benchmark.                                                                                                                                                   |

## 2. Metadata

| Attribute                      | Value                                                                                                                                                                                                                                           |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Base benchmarks extended       | [[MMLU]] (57 subjects, ~14K multiple-choice problems), [[GSM8K]] (8.5K grade-school math word problems), [[AgentBench]] os-std subset (multi-turn Linux/OS interaction tasks, ~8 turns/problem on average)                                      |
| Input/output format            | MMLU: multiple-choice question → letter/answer choice. <br>GSM8K: math word problem → exact numerical answer. <br>AgentBench os-std: multi-turn natural-language task → console commands/outputs, evaluated on exact console output correctness |
| Rounds per experiment          | 4 total (1 Standard round + 3 LLM-Evolve rounds)                                                                                                                                                                                                |
| Models evaluated               | 8 LLMs: Llama2-7B, Llama2-70B, Llama3-8B, Llama3-70B, Mistral-7B-v0.2, Qwen2-72B, GPT-3.5-turbo, GPT-4                                                                                                                                          |
| Retriever used (main results)  | Contriever (dense retriever); BERT used in ablation                                                                                                                                                                                             |
| Generation temperature         | 0 (fixed, to remove randomness)                                                                                                                                                                                                                 |
| Compute                        | 7B/8B models: 1×A100 GPU; 70B models: 8×A100 GPUs; ~1 hour per LLM-Evolve round                                                                                                                                                                 |
| Feedback source (main results) | Ground-truth benchmark labels; LLM-as-feedback (Llama3-70B) tested only in ablation                                                                                                                                                             |

## 3. What it Measures (What)

### 3.1 Task Definition

LLM-Evolve does not introduce new test items; instead it redefines the _evaluation protocol_ over existing benchmarks to measure a different capability: **whether an LLM can improve its own benchmark accuracy over sequential rounds by retrieving and reusing its own past successful (input, output) experiences as few-shot demonstrations**, in place of the benchmark's fixed few-shot prompts.

The measured quantity in each round is standard task accuracy (multiple-choice accuracy for MMLU, exact-match numerical accuracy for GSM8K, task-completion accuracy for AgentBench os-std), but tracked _across rounds_ rather than as a single score.

### 3.2 Example Records

Using the paper's running MMLU example (Table 1, Llama3-8B):

- **Round 0 (Standard):** Llama3-8B answers all 57-subject MMLU questions using the benchmark's fixed few-shot demonstrations → 65.35% accuracy.
- **Round 1:** Correctly-answered round-0 questions (f_i = True) are embedded and stored in 𝒟. For each new query, the top-k nearest stored questions (by Contriever embedding distance) replace the fixed few-shot prompt → 70.36% accuracy.
- **Round 2–3:** 𝒟 is refreshed with round-1/round-2 correct experiences and reused → 71.02%, then 71.08% accuracy (diminishing gains after round 1).

### 3.3 Metrics

|Metric|Definition|
|---|---|
|Round-wise accuracy|Standard task accuracy metric of the underlying benchmark (multiple-choice accuracy for MMLU, exact-match for GSM8K, task-completion accuracy for AgentBench os-std), computed independently at each round.|
|LLM-Evolve Gain|Difference between the highest-accuracy LLM-Evolve round and the Standard (round 0) accuracy, reported per model per benchmark (e.g., Llama2-7B on MMLU: 5.74 points).|

## 4. Uniqueness (Why)

### 4.1 Other Benchmarks

|Benchmark|Relationship to LLM-Evolve|
|---|---|
|Winogrande (Sakaguchi et al., 2021)|Also derived from an existing benchmark (Winograd), but modifies the _test set_ to reduce annotation bias, rather than modifying the _evaluation protocol_ while keeping the test set fixed.|
|MMLU-Redux (Gema et al., 2024)|Re-annotates a 3,000-question subset of MMLU to correct labeling errors; changes ground truth, not evaluation protocol.|
|MMLU-Pro (Wang et al., 2024)|Extends MMLU with harder, more reasoning-focused questions; changes the test set itself.|
|MT-bench (Zheng et al., 2024)|A multi-turn benchmark independently designed for conversational evaluation; LLM-Evolve borrows its multi-turn structure conceptually for extending to multi-turn settings but does not evaluate on it directly.|

### 4.2 Uniqueness

Unlike prior derivative benchmarks, LLM-Evolve makes **no changes to the underlying test sets or metrics** of MMLU, GSM8K, or AgentBench. Its novelty lies entirely in the _evaluation protocol_: transforming single-shot evaluation into a sequential, feedback-driven, memory-augmented setting.
This preserves direct comparability with previously published benchmark results while enabling measurement of an LLM's capacity to self-improve from its own interaction history 

## 5. Generation Method (How)

LLM-Evolve is constructed as a wrapper around existing benchmarks rather than generating new data. The process, threaded through the MMLU/Llama3-8B example:

> **Running example (Llama3-8B on MMLU):**
> 
> 1. **Round 0 (standard evaluation):** Llama3-8B is evaluated on all MMLU questions using the benchmark's original fixed few-shot demonstrations, $y^{lm} = p_θ(x, \{x^{demo}, y^{demo}\})$. Accuracy: 65.35%.
> 2. **Memory construction:** For every question $x^{lm}_i$, the model's own output $y^{lm}_i$ and correctness feedback $f_i$ (from the MMLU ground-truth key) are stored. Only positive experiences (f_i = True) are kept in 𝒟; incorrect ones are discarded.
> 3. **Retrieval for round 1:** For each new query x, Contriever embeds x and all stored $x^{lm}_j$; the top-k closest positive experiences (by L2 distance in embedding space) replace the original fixed demonstrations: $y^{LLM-Evolve} = p_θ(x, \{x^{lm}_i, y^{lm}_i\})$.
> 4. **Iteration:** After round 1 produces new outputs, 𝒟 is refreshed with round-1's correct experiences, and the same retrieval process is repeated for rounds 2 and 3, each time drawing on an expanded, more recent memory.
> 5. **Multi-turn adaptation (AgentBench):** For multi-turn tasks, full trajectories $(x^{lm}_{i1}, y^{lm}_{i1}, ..., x^{lm}_{it}, y^{lm}_{it}, f_i)$ are stored as single units, and retrieval is keyed on similarity of the first-turn input only, so an entire successful past trajectory, not just one turn, is reused as the demonstration.

This design requires no new annotation or data generation; it only requires:
- a scoring function to obtain f_i (ground truth or LLM judge) 
- a dense retriever to index and query 𝒟.

## 6. Gaps

- The framework only retains **positive-feedback experiences**; the paper's own ablation shows retrieval and negative-experience masking variants were tried and found inferior, but these are only briefly mentioned, not deeply analyzed (author-stated).
- Main results rely on **ground-truth benchmark labels** for feedback, which the authors themselves note is unrealistic for real-world deployment where such labels are unavailable (author-stated, in Limitations).
- Only **three benchmarks and eight models** are tested; broader benchmark/model coverage is left for future work (author-stated).
- No analysis is given o**f how memory size/staleness or retrieval latency would scale** in longer-horizon or continuously-running deployments beyond 3–4 rounds.
- The paper does not examine **whether performance gains generalize to genuinely novel problems** dissimilar from anything in memory, versus benefiting mainly from near-duplicate or highly similar past questions being retrieved.

## 7. Highlights

> [!PDF|255, 208, 0] [[LLM-Evol.pdf#page=1&annotation=496R|LLM-Evol, p.16937]]
> > ose a different approach: modifying the settings of established benchmarks, such as MMLU, without changing their test sets and metrics

> [!PDF|255, 208, 0] [[LLM-Evol.pdf#page=1&annotation=499R|LLM-Evol, p.16937]]
> > Under LLM-Evolve, LLMs are assessed across multiple rounds, where the environment will provide feedback on each round’s outcomes to inform subsequent LLM evaluations

> [!PDF|255, 208, 0] [[LLM-Evol.pdf#page=2&annotation=502R|LLM-Evol, p.16938]]
> > Ms can gradually achieve better benchmark performance by using better fewshot IO pairs based on their evolving interaction histories

