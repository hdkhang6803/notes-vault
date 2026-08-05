---
Tags:
Date: "2024"
Authors: Zuxin Liu, Thai Hoang, Jianguo Zhang, Ming Zhu, Tian Lan, Shirley Kokane, Juntao Tan, Weiran Yao, Zhiwei Liu, Yihao Feng, Rithesh Murthy, Liangwei Yang, Silvio Savarese, Juan Carlos Niebles, Huan Wang, Shelby Heinecke, Caiming Xiong
Venue: NeurIPS
Paper: "APIGen: Automated PIpeline for Generating Verifiable and Diverse Function-Calling Datasets"
---
# 1. Terminology

|Term|Definition|
|---|---|
|APIGen (Automated PIpeline for Generating verifiable and diverse function-calling datasets)|The end-to-end data generation framework that samples APIs and seed query-answer (QA) pairs, prompts an LLM to synthesize new QA pairs, and passes each candidate through a three-stage verification process before adding it back to the seed pool.|
|BFCL (Berkeley Function-Calling Leaderboard)|An external evaluation benchmark of 1,700 test cases spanning Java, JavaScript, and Python API calls, scored via Abstract Syntax Tree (AST) evaluation and Executable Function evaluation across simple, multiple, parallel, and parallel-multiple categories plus relevance detection.|
|Format Checker (Stage 1)|The first verification stage that discards a generated data point unless its output strictly contains parseable "query," "answer" (and optional "thought") JSON fields, and unless every function name/argument used in the answer exists in the provided API set.|
|Execution Checker (Stage 2)|The second verification stage that runs each well-formatted function call against its real backend (Python functions imported and executed in a subprocess, or REST endpoints called over HTTP) and discards calls that fail to execute, logging the specific error type (e.g., argument type error, timeout, missing argument).|
|Semantic Checker (Stage 3)|The third verification stage, an LLM judge that is given the available functions, the generated query, the function call(s), and the Stage 2 execution results, and outputs a yes/no pass decision assessing whether the call's target, arguments, call count, and results semantically match the query's intent.|
|API Sampler|The sampling module that extracts one or more function descriptions from the API library and standardizes them into the uniform JSON schema used throughout the pipeline.|
|Example Sampler|The sampling module that draws a variable number of seed QA examples per query-diversity category to serve as few-shot references for the generator LLM.|
|Prompt Sampler|The sampling module that selects from a library of prompt templates (spanning concise to ambiguous/misspelled query framings) to steer the style of generated queries.|
|Query style categories (Simple, Multiple, Parallel, Parallel Multiple)|Four data-generation categories distinguished by how many APIs are offered and how many function calls the correct answer requires: Simple (1 API, 1 call), Multiple (several APIs, pick 1), Parallel (1 API, several calls in one query), Parallel Multiple (several APIs, several calls each, possibly repeated).|
|xLAM (large action model)|The family of function-calling models (xLAM-1B and xLAM-7B, fine-tuned from DeepSeek-Coder-1.3B/7B-instruct) trained on the APIGen-generated dataset using the AgentOhana training pipeline.|
|Relevance detection data|Auxiliary training examples (8,000 points) constructed by dropping tools or required arguments from existing generated examples and relabeling the answer as an empty tool call, teaching the model to refuse when no available tool can satisfy the query.|

# 2. Method Summary (What)

APIGen is an automated pipeline that synthesizes function-calling training data (query + available-tools + answer triples) at scale from a library of 3,673 real, executable APIs (3,539 REST APIs cleaned from ToolBench plus 134 well-documented Python functions) across 21 categories. For each generation round, it samples one or more APIs, a few seed QA examples, and a prompt template targeting one of four query-style categories, then asks an LLM to produce new query-answer pairs in a standardized JSON schema. Every candidate answer is then filtered through a three-stage verification pipeline (format → execution → semantic) before being accepted into the released dataset and fed back as a seed example for future generation rounds. The authors release 60,000 verified entries and use them to fine-tune two small function-calling models, xLAM-1B and xLAM-7B, which are evaluated on the Berkeley Function-Calling Leaderboard.

# 3. What it Solves (Why)

- Existing function-calling training datasets are largely static and lack rigorous verification, leading to inaccuracies that hurt fine-tuned model performance in real-world use.
    - ToolBench and Toolalpaca generate tool-use data via ChatGPT-written documentation/descriptions without executing the calls against real backends, so incorrect or hallucinated calls can enter the dataset unchecked.
    - AgentInstruct, AgentOhana, and Lumos unify multiple existing agent datasets and training pipelines but inherit the noise of their unverified source data.
    - Public datasets for the "parallel" function-calling query style (multiple concurrent calls to answer one query) are essentially absent, leaving models undertrained on this common real-world pattern.
    - Models trained on narrow, single-domain APIs struggle to generalize to unseen API domains at deployment time.
    - => APIGen's proposed solution: because function-calling answers, unlike open-ended chat responses, can be _directly executed_ against their APIs, the pipeline exploits this property with a three-stage verification process (format checking → real execution → LLM-based semantic alignment checking) applied to a large, diverse (3,673 API, 21-category) executable API library, explicitly generating underrepresented query styles like parallel and parallel-multiple calls.

# 4. Methodology (How)

**System / Data structure**

- All APIs, function calls, and generator outputs are represented in one standardized JSON schema (name, description, parameters with type/description/default/required fields), which lets APIGen ingest new API sources (REST or Python) via lightweight format converters without touching the rest of the pipeline.
- The API library itself was built by cleaning ToolBench's 16,464 RapidAPI REST APIs down to 3,539 executable, well-documented ones (via data-quality filtering, live accessibility testing, and docstring regeneration) and consolidating overlapping categories (e.g., merging "Finance" and "Financial") into 21 categories, then adding 134 curated Python functions.

**Generation flow**

1. Sample one or more APIs (API Sampler), seed QA examples for the target query style (Example Sampler), and a prompt template (Prompt Sampler).
2. Fill the sampled content into a query-style-specific prompt (e.g., the "parallel" generation prompt instructs the LLM to produce queries containing multiple independent sub-requests).
3. Call the generator LLM (e.g., DeepSeek-V2-Chat, DeepSeek-Coder-33B-Inst, Mixtral-8x22B-Inst, Mixtral-8x7B-Inst) to batch-produce several query-answer JSON pairs in one inference call, reducing token cost.

**Verification flow (Multi-Stage Data Verification)**

1. **Format Checker (Stage 1):** Discard outputs that don't parse into the required "query"/"answer" JSON fields, or whose function calls reference arguments/functions not present in the supplied API set.
2. **Execution Checker (Stage 2):** Execute every well-formatted call against its real backend (subprocess import for Python functions, live HTTP call for REST APIs); discard calls that error out, recording the failure type (type error, timeout, missing argument, etc.).
3. **Semantic Checker (Stage 3):** Pass the query, the call(s), the available functions, and the Stage 2 execution results to a separate LLM judge, which checks intent alignment, correct function/argument selection, correct call count, and whether results are error-free and relevant; only "yes" verdicts pass.
4. Data points that pass all three stages are added to the release set **and** fed back into the seed pool to diversify future generation rounds.

> **Running example (weather query, threaded through the pipeline):** The API Sampler pulls `weather_api.get_current_weather(location, units)` from the library. The Example Sampler and Prompt Sampler assemble a "simple" style prompt. The generator LLM produces the query _"What is the weather in Palo Alto?"_ with answer `get_current_weather(location="Palo Alto", units="Celsius")`. Stage 1 confirms the JSON is well-formed and that `location`/`units` are valid parameters of the API. Stage 2 actually calls the weather backend with these arguments and confirms it returns a valid result rather than an error. Stage 3 checks that returning today's Palo Alto weather in Celsius genuinely answers the user's query; since it does, the example passes and is added to the training set (and back into the seed pool for generating further weather-style examples).

# 5. Benchmarks

## 5.1 Other Baselines

|Baseline|Description|
|---|---|
|GPT-4 family (GPT-4o, GPT-4-Turbo, GPT-4-0125-Preview)|Closed-source frontier LLMs evaluated in both prompted and native function-calling (FC) modes.|
|Claude-3 family (Opus, Haiku, 3.5-Sonnet)|Closed-source Anthropic models compared in both prompt-based and FC modes.|
|Gemini-1.5-Pro / Flash|Closed-source Google models evaluated in FC mode.|
|Llama3-70B-Instruct, Mixtral/Mistral-large|Open-weight large models evaluated via prompting or FC mode.|
|Gorilla-OpenFunctions-v2, Command-R-Plus, Nexusflow-Raven-v2, DBRX-Instruct|Open-source function-calling-focused baselines without public verified training data.|
|DeepSeek-Coder-1.3B / 7B-instruct-v1.5 (base models)|The un-fine-tuned base checkpoints from which xLAM-1B and xLAM-7B are trained, used to isolate the effect of APIGen data (base 7B model ranks only 45th on BFCL).|

## 5.2 Benchmarks

|Benchmark|Role|
|---|---|
|Berkeley Function-Calling Leaderboard (BFCL)|Primary evaluation benchmark; 1,700 test cases across Java/JavaScript/Python scenarios, scored on AST evaluation (syntactic correctness of calls) and Executable Function evaluation (real execution correctness), plus a relevance-detection category.|
|Internal ablation (Fig. 5)|Re-adds Stage-2-filtered and Stage-3-filtered ("failed") data back into training to measure the verification pipeline's own contribution to downstream accuracy.|
|Human evaluation (600 sampled data points, 3 annotators)|Manually checks parameter-value accuracy and appropriateness of call count on a sample of the released dataset.|

## 5.3 Notable Results

- xLAM-7B (FC) ranks 3rd overall on the BFCL leaderboard (88.24% accuracy), surpassing GPT-4o, GPT-4-Turbo, Gemini-1.5-Pro, and several Claude-3 models, despite its much smaller (7B) parameter count.
- xLAM-1B (FC) ranks 25th (78.94% accuracy), outperforming GPT-3.5-Turbo, Claude-3 Haiku, Command-R-Plus, DBRX-Instruct, and Mistral-large — notably from a 1B-parameter model.
- The ablation shows that reintroducing Stage-2/Stage-3-filtered ("low-quality") data into training harms BFCL accuracy by roughly 4–6 points for xLAM-7B and roughly 6–12 points for xLAM-1B, with the smaller model more sensitive to unfiltered data.
- Human evaluation found only 28 of 600 sampled data points (≈4.7%) had minor issues, i.e., ~95.3% of sampled data was judged high quality.
- Pass rates from raw generation varied substantially by generator LLM strength: DeepSeek-V2-Chat (236B) achieved an 84.15% pass rate versus 34.42% for DeepSeek-Coder-33B-Inst, indicating stronger generator models produce more verifiable data.

# 6. Strengths

- **Execution-grounded verification:** Uniquely exploits the fact that function calls (unlike open-ended chat) can be directly executed, letting the pipeline catch incorrect/hallucinated calls that a purely LLM-judged pipeline would miss.
- **Demonstrated small-model efficiency:** Shows that data quality/diversity can substitute for model scale — a 7B and even a 1B model outperform several much larger closed-source frontier models on a comprehensive external leaderboard.
- **Fills a coverage gap:** Explicitly targets parallel and parallel-multiple query styles that the authors identify as rarely present in public function-calling datasets.
- **Format-agnostic scalability:** The standardized JSON schema lets new API sources (REST, Python, and by extension future formats) be incorporated via lightweight converters without redesigning the core pipeline.
- **Validated at multiple levels:** Combines automated benchmark evaluation, an internal filtering ablation, and independent human evaluation (600 samples, 3 annotators) to corroborate dataset quality claims.

# 7. Gaps

- **Single-turn only (author-stated):** The framework and released dataset currently only generate single-turn function-calling examples, not multi-turn agent-tool-human interactions.
- **Limited API modalities (author-stated):** Only REST APIs and Python functions are supported; other tool/API paradigms (e.g., GUI actions, databases, multimodal tools) are not covered.
- **Imperfect semantic checker (author-stated):** The paper notes the final semantic-checking stage "cannot guarantee correctness," relying on execution feedback to improve but not assure judgment accuracy.
- **Generator-dependent quality (Claude-proposed):** Pass rates varied from ~34% to ~84% depending on the generator LLM, suggesting dataset quality and diversity may be sensitive to which (and how many) generator models are used — a dependency the paper does not deeply analyze beyond reporting the rates.
- **Checker LLM bias/circularity (Claude-proposed):** Since both query generation and semantic verification rely on LLMs, systematic blind spots or biases shared between generator and checker LLMs could pass through undetected, an angle not explored in the paper's evaluation.
- **Category/domain skew (Claude-proposed):** The 21-category API distribution (Fig. 4) is uneven (e.g., Data and Finance are large slices while Music and Tools are small), which could bias downstream model competence toward better-represented domains, though this is not directly analyzed in the results.

# 8. Highlights

_(left blank)_