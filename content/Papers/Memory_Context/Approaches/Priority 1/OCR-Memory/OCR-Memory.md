---
Tags:
Date: "2026"
Authors: " Jinze Li, Yang Zhang, Xin Yang, Jiayi Qu, Jinfeng Xu, Shuo Yang, Junhua Ding, Edith Cheuk-Han Ngai"
Venue: ACL
Paper: "OCR-Memory: Optical Context Retrieval for Long-Horizon Agent Memory"
Memory type:
  - Token-level
Agent env: 2-model pipeline
Record format: Image
Memory architecture:
Tackle Module: Memory retrieval
Need offline initialization: true
Fine-tuning?: false
Other tags:
---
# 1. Terminology

| Term                                          | Definition                                                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Long-Horizon Agent**                        | An LLM-based agent that handles continuous, multi-step tasks over extended periods (e.g., browsing the web across hundreds of steps), where success depends on memory of past actions                              |
| **Trajectory (τ)**                            | The full sequence of an agent's actions in one episode (user messages, reasoning steps, tool calls, and tool results, essentially the agent's "interaction log")                                                   |
| **Text-RAG (Retrieval-Augmented Generation)** | The classic memory approach: store past text in a database, embed it with dense vectors, and retrieve the most similar chunks when a new query arrives                                                             |
| **Optical Context Retrieval**                 | The core idea of the paper: instead of storing and retrieving text, store interaction history as _images_ and use a vision model to read them. Analogous to taking a photo of your notes instead of re-typing them |
| **Visual Token**                              | A token produced by encoding an image through a vision encoder; a single image can be represented by far fewer visual tokens than its equivalent raw text would consume                                            |
| **Set-of-Mark (SoM) Prompting**               | A visual grounding technique where numbered red bounding boxes are overlaid on each text segment in the image, turning "find the relevant text" into "predict the index number of the relevant box"                |
| **Locate-and-Transcribe**                     | OCR-Memory's two-step retrieval: (1) _locate_ the relevant segment index from the image, then (2) _transcribe_ (fetch) the exact original text from the log using that index (no free-form generation)             |
| **Hallucination**                             | In retrieval, this means "recalling" evidence that was never actually stored                                                                                                                                       |
| **Active Recall Upscaling**                   | When a low-resolution (old) memory image is identified as relevant, it is restored to full resolution on demand, mimicking how humans can vividly recall a fuzzy memory once reminded of it                        |
| **Age-Aware Adaptive Resolution**             | Older memory images are progressively downsampled to lower resolutions to save visual tokens, analogous to how human memory becomes "fuzzy" over time                                                              |
| **Weighted Binary Cross-Entropy (BCE)**       | The training loss used to teach the model which segments are relevant; positive (relevant) examples are penalized more heavily for being missed (higher w+) to encourage recall                                    |
| **DeepSeek-OCR**                              | The 3B-parameter vision-language backbone used in OCR-Memory, pre-trained for optical compression of dense documents into compact visual tokens                                                                    |
| **HotpotQA**                                  | A multi-hop question-answering dataset used here for fine-tuning: the model learns to select supporting paragraphs from a set, providing a proxy for agent-history retrieval                                       |
| **Needle-in-a-Haystack (NIAH)**               | A benchmark where a specific "needle" fact is hidden inside a long document; tests whether a retrieval system can find it regardless of context length                                                             |
| **RULER Benchmark**                           | A long-context evaluation suite; the NIAH test from RULER is used here to stress-test OCR-Memory's scalability up to 32k context lengths                                                                           |
| **Mind2Web**                                  | A web-navigation agent benchmark; tasks involve multi-step browser interactions, evaluated on Element Accuracy, Action F1, Step SR, and Task SR                                                                    |
| **AppWorld**                                  | An API-interaction agent benchmark with Easy/Medium/Hard difficulty levels based on how much historical backtracking is needed                                                                                     |
| **Recall@K**                                  | Retrieval metric: what fraction of the time does the correct segment appear in the top-K retrieved results?                                                                                                        |
| **MRR (Mean Reciprocal Rank)**                | Retrieval metric: the average of 1/rank of the first correct result; higher is better                                                                                                                              |
| **Compression Ratio**                         | Visual tokens used vs. raw text tokens that would have been needed; a 10× ratio means the image uses 10× fewer tokens than the equivalent text                                                                     |
| **Episode**                                   | One complete task interaction: the agent receives a goal, executes steps, and terminates -> producing one trajectory                                                                                               |

---
# 2. Paper Summary (What)

OCR-Memory is a novel **agent memory framework** that reimagines _where_ and _how_ an LLM agent stores and retrieves its interaction history. Instead of keeping histories as text (which is expensive in tokens) or summarizing them (which loses detail), it:

1. **Stores** past interaction trajectories as **images** (screenshots/renders of the logs) in a Visual Memory Bank.
2. **Retrieves** relevant history via a **Locate-and-Transcribe** pipeline: a vision model scans the images, identifies the relevant numbered segments, and then deterministically fetches the original verbatim text without generating anything from scratch.
3. **Manages memory age** by keeping recent memories at high resolution and progressively compressing older ones into low-resolution thumbnails, with on-demand upscaling when they become relevant again.

The framework is built on DeepSeek-OCR (3B), fine-tuned with LoRA on HotpotQA for discriminative retrieval, and achieves state-of-the-art performance on both Mind2Web and AppWorld under strict token budgets.

---

# 3. What It Solves (Why)

| Approach                   | How it works                                               | Problem                                                                      |
| -------------------------- | ---------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Raw Text Storage**       | Keep the full trajectory text and inject it                | Too many tokens; quickly blows the budget                                    |
| **Text Summarization**     | Compress episodes into short summaries                     | Loses fine-grained detail (exact error messages, step order, tool outputs)   |
| **Text-RAG**               | Embed text as vectors, retrieve top-K chunks by similarity | Brittle: topically similar ≠ causally relevant; noisy retrieval; token-heavy |
| **Experience Abstraction** | Distill trajectories into reusable skills/rules            | Discards low-level specifics needed for debugging or multi-step grounding    |
- A 4,000-token trajectory rendered as an image → 256-400 visual tokens (≈16× compression)
- At retrieval, only 64-256 visual tokens are needed to scan the image for the relevant segment
- The actual text is fetched verbatim from a log, costing 0 extra tokens to "generate"

This allows an agent with a 4,096-token budget to effectively consult arbitrarily long histories.

---

# 4. Methodology (How)
![[Pasted image 20260520013445.png]]

> **Running Example:** An agent is helping a user manage a Hugging Face model hub. It has already completed 20 episodes over the past hour. 

## 4.1. Memory Architecture

Each stored memory item `mᵢ` is a triple:


```
mᵢ = ( Iᵢ ,  {s_{i,1}, s_{i,2}, ..., s_{i,K}} ,  πᵢ )
      ───    ──────────────────────────    ───
    image      verbatim text segments    metadata
   (visual      (stored in a separate    (timestamp,
   memory)        deterministic log)     episode id)
```
- **`Iᵢ` : the SoM-annotated image:** The interaction log of episode `i` is rendered into an image where every text segment gets a numbered red bounding box. This is what the retriever _sees_ at query time.
- **${s_{i,k}}$ : the verbatim text log:** The original raw text is kept separately in a structured database. This is what gets _fetched_ after retrieval (the retriever never reads this directly).
- **`πᵢ`: metadata:** Timestamps and episode IDs that drive the aging policy.

For our example, after 20 completed episodes, Episode 15's memory item looks like:

```
m₁₅:
  ┌── Iᵢ (image, SoM-annotated) ─────────────────────────┐
  │  [1] Specifying a dataset                             │
  │      You can specify the datasets used to train...    │
  │  [2] Specifying a task (pipeline_tag)                 │
  │      You can specify the pipeline_tag in the model... │
  │  [3] Specifying a license                             │
  │      You can specify the license in the model card... │
  └───────────────────────────────────────────────────────┘
  {sᵢ,k}: { "1": "Specifying a dataset...",
             "2": "Specifying a task (pipeline_tag)...",
             "3": "Specifying a license..." }   ← verbatim log
  πᵢ:     { timestamp: 55 min ago, episode_id: 15 }
```

**The aging policy** determines what resolution `Iᵢ` is stored at. The rule is simple: the 5 most recent episodes stay at full fidelity; everything older is a thumbnail:

```
Visual Memory Bank state before our query:
══════════════════════════════════════════════════════
  Episode  1–15   │  512×512   │  64 visual tokens  │ aged
  Episode 16–20   │ 1024×1024  │ 256 visual tokens  │ recent
══════════════════════════════════════════════════════
```

This is the "vivid-to-fuzzy" property of human memory. 
Crucially, the raw text logs `{sᵢ,k}` are **always** preserved at full fidelity regardless of image resolution.

---
## 4.2. Retrieval and Updates
### 4.2.1. Step 1: Retrieve (Locate) - Scan All Images for Relevant Segments

> A new query arrives: _"How do I specify a task for my model?"_

The **Optical Retriever** (DeepSeek-OCR 3B, fine-tuned) takes the query and scans every stored image `Iᵢ` in the bank. For each image, it outputs a **binary relevance vector** (predicting only `0` or `1` per segment):

```
Query: "How do I specify a task for my model?"

Scanning episode 15's image (512×512, aged thumbnail):
  Segment [1] → p(15,1) = 0.08  →  0  (datasets, not relevant)
  Segment [2] → p(15,2) = 0.91  →  1  ← RELEVANT (pipeline_tag)
  Segment [3] → p(15,3) = 0.12  →  0  (licenses, not relevant)

Relevance vector:  ŷ_15 = [0, 1, 0]
```

The probability `p(i,k)` is derived from the decoder's token logits for `"1"` vs `"0"` at each segment position. 
![[Pasted image 20260514170553.png]]
A recall-oriented selection rule picks any segment where $\rho > \tau = 0.4$, with a Top-5 fallback guaranteeing at least 5 candidates per image even when confidence is low. After scanning all 20 episodes:
![[OCR-Memory.pdf#page=4&rect=312,105,532,155&color=yellow|OCR-Memory, p.4]]

```
Global index set:  Ŝ(q) = { (15, 2) }
                           ───────────
                           episode 15, segment 2
```

---

### 4.2.2. Step 2: Retrieve (Transcribe)- Fetch Verbatim Text from the Log

With the relevant index `(15, 2)` confirmed, the system performs a **deterministic lookup** into the verbatim text log:

```
Fetch( (15, 2) )  →  s_{15,2}
  = "Specifying a task (pipeline_tag): You can specify the pipeline_tag
     in the model card metadata. The pipeline_tag indicates the type of
     task the model is intended for. This tag will be displayed on the
     model page and users can filter models on the Hub by task."
```

No language model ever generates this text.

---
### 4.2.3. Step 3: Retrieve (Active Recall): Restore Faded Memories on Demand

if the retrieved memory's image is in low-fidelity, OCR-Memory triggers **Active Recall Upscaling**: discard the low-fidelity image and put that memory text through image rendering pipeline (See step 5)

```
m_{15} image before:  512×512   →  Discarded
        ↓  re-render from {sᵢ,k} text log
m_{15} new image :   1024×1024  →  pinned at high-res for the rest of this episode
        (exempt from further aging decay)
```

The high-res version is reconstructed on the fly and cached in an active visual cache for the duration of the current episode.

---
### 4.2.4. Step 4: Inject Evidence and Run the Reasoning Agent

The fetched verbatim text is injected into the primary reasoning agent's (GPT-4) context window alongside the current query:

```
┌────────────────────────────────────────────────────────────┐
│  Agent Prompt (GPT-4, 4096-token budget)                   │
│                                                            │
│  [Retrieved Evidence — 596 tokens]:                        │
│    "Specifying a task (pipeline_tag): You can specify      │
│     the pipeline_tag in the model card metadata..."        │
│                                                            │
│  [Current Query]:                                          │
│    "How do I specify a task for my model?"                 │
└────────────────────────────────────────────────────────────┘

GPT-4 Response:
  "Add `pipeline_tag: text-classification` to your model
   card YAML metadata to specify the task type."
```

---
### 4.2.5. Step 5: Store the New Episode into the Memory Bank

After the agent completes this episode, its new interaction log is written back into the bank. 

**5a. Render with SoM**: The log is segmented and rendered into a fresh SoM-annotated image at full resolution:

```
Episode 21 log:
  s_{21,1}: "User asked how to specify a task for a model."
  s_{21,2}: "Agent retrieved pipeline_tag documentation."
  s_{21,3}: "Agent advised: pipeline_tag: text-classification."

  ↓  Render with red bounding boxes, 36pt bold index labels

  ┌──────────────────────────────────────────────────────┐
  │  [1] User asked how to specify a task for a model.   │
  │  [2] Agent retrieved pipeline_tag documentation.     │
  │  [3] Agent advised: pipeline_tag: text-class...      │
  └──────────────────────────────────────────────────────┘
     I_{21} — stored at 1024×1024 (fresh, high-res)
```

**5b. Apply Aging Policy**: Episode 16, previously in the recency window, is now the 6th-most-recent and gets downsampled. Episode 15 (active-recalled this episode) remains pinned:

```
Updated Visual Memory Bank after Episode 21:
══════════════════════════════════════════════════════════
  Episode  1–14   │  512×512   │  64 visual tokens  │ aged
  Episode 15      │ 1024×1024  │ 256 visual tokens  │ pinned (active recalled)
  Episode 16      │  512×512   │  64 visual tokens  │ newly aged
  Episode 17–21   │ 1024×1024  │ 256 visual tokens  │ recent
══════════════════════════════════════════════════════════
```

The verbatim logs `{s_{21},k}` are stored in the deterministic text database alongside `I_{21}`. The full cycle is now complete, episode 21 is ready to be retrieved by any future query.

---
## 4.3. Fine-Tune the Optical Retriever (Offline, One-Time)

The retriever backbone (DeepSeek-OCR 3B) is fine-tuned **offline** before deployment, on **HotpotQA** repurposed as a visual retrieval task:
- 10 Wikipedia paragraphs are rendered into a SoM-annotated image
- Ground-truth supporting facts act as binary labels (relevant = 1, irrelevant = 0)
- **Weighted BCE loss** with `w+ = 2.0 > w− = 1.0` biases toward recall — missing a relevant segment is penalized more than a false positive
![[Pasted image 20260514234703.png]]
- Vision encoder θ_vis is **frozen**; only the language decoder is updated via **LoRA** (rank=16, α=32), preserving optical recognition while teaching discriminative retrieval
- **Resolution curriculum**: each training image is randomly downsampled to 512×512 or 1024×1024 with probability [0.3, 0.7], training the model to retrieve reliably from blurry aged thumbnails as well as sharp recent ones

```
HotpotQA instance → rendered as SoM image (10 paragraphs, 10 boxes)
Query: "Who co-founded Apple with Steve Wozniak?"
Labels: y = [0, 0, 1, 0, 0, 0, 1, 0, 0, 0]
		           ↑              ↑
           paragraph about      paragraph about
              Apple Inc.          Steve Jobs

Loss: Weighted BCE  →  w₊ = 2.0,  w₋ = 1.0
      (missing a relevant segment penalized 2× more than a false positive)
```
The result is a model that has shifted from "read and transcribe everything" (pre-trained OCR behaviour) to "read and select only what this query needs" (discriminative retrieval).

---

# 5. Strengths

- **Substantial Token Reduction in the Primary Reasoning Agent's Context** OCR-Memory reduces tokens injected into GPT-4 from 3,980 → 596 per step (6.7×). This is a meaningful practical gain: GPT-4's context window is the scarcest resource in long-horizon agent systems, and bloating it with retrieved text degrades reasoning quality. The advantage compounds as the budget tightens 

```
Text-RAG:
  Retrieval cost  → cheap  (embedding cosine sim, 0 LLM tokens)
  Injection cost  → expensive  (3,980 raw text tokens into GPT-4)

OCR-Memory:
  Retrieval cost  → expensive  (N × visual tokens in DeepSeek-OCR,
                                1.7s latency, 1.47 MB disk per episode)
  Injection cost  → cheap  (596 verbatim tokens into GPT-4)
```

- **Relation-Aware Retrieval** By rendering multiple segments into a single image, the optical retriever reads all segments simultaneously within one forward pass, enabling it to reason about cross-segment relationships 
- **Dynamic Resolution Achieves Near High-Res Accuracy at Near Low-Res Cost** The two-tier resolution strategy achieves 46.1% Step SR against 46.5% for static high-resolution, while consuming only 82 average visual tokens per frame versus 256 — a 3× token saving in the retrieval model with negligible accuracy loss.

---

# 6. Gaps
- **"Hallucination-Free Retrieval" is a Misleading Claim** The paper's central marketing claim that OCR-Memory achieves hallucination-free retrieval conflates two structurally different failure modes. The locate step is performed by a fine-tuned LLM (DeepSeek-OCR) that can and does predict wrong indices (Recall@1 = 78.6% means 21.4% of individual retrievals select the wrong segment). The honest claim is fabrication-free transcription which is a real and meaningful property, but substantially narrower than what the paper asserts.
- **Retrieval Cost Is Not Globally Cheaper, Only Redistributed** The paper frames OCR-Memory as token-efficient, but this holds only for the primary reasoning agent's context. The retrieval model (DeepSeek-OCR) must process every stored image sequentially per query and is strictly more expensive than cosine similarity on text embeddings, which operates in embedding space with zero LLM token consumption. The reported 1.7s retrieval latency versus 0.3s for Text-RAG confirms this overhead.
- **High Storage Footprint with No Mitigation Strategy** Each episode requires 1.47 MB of disk storage versus 18 KB for text-RAG — an 82× increase — arising from storing SoM-annotated images alongside verbatim text logs.
- **Training Overhead Limits Accessibility**: Unlike training-free retrieval baselines, OCR-Memory requires fine-tuning a 3B vision-language model with LoRA, a resolution curriculum, and a weighted BCE objective on a repurposed version of HotpotQA. 
- **Severe Domain Gap Between Training Data and Deployment** The retriever is fine-tuned on HotpotQA, a dataset of encyclopedic multi-hop questions over Wikipedia paragraphs. It is then deployed on Mind2Web (browser interaction logs containing HTML snippets, UI element descriptions, and navigation traces) and AppWorld (API call logs with JSON payloads, error messages, and function signatures)
- **Real Multimodal Agent Histories Are Not Addressed** OCR-Memory renders only text logs as images. Modern GUI agents (e.g., CogAgent, AppAgent) operate on actual screen captures that already encode rich visual information that is absent from text renders. 
	=> Store real screenshots directly as memory items, which would preserve modality-native information rather than converting everything to text first and then back to an image.

---

# 7. Benchmarks

### 7.1.1. Baselines

| Baseline                                                | Paradigm               | Key Characteristics                                                                                  |
| ------------------------------------------------------- | ---------------------- | ---------------------------------------------------------------------------------------------------- |
| **Zero-Shot**                                           | No memory              | Agent operates with no historical context; performance lower bound                                   |
| **Dense Text-RAG** (Retrieval)                          | Retrieval-based        | Embeds text chunks as dense vectors; retrieves top-K by cosine similarity at inference time          |
| **[[MemoryBank]]** (Zhong et al., 2024)                 | Retrieval-based        | Maintains a structured long-term text memory bank; supports time-aware decay and update              |
| **[[AWM]] - Agent Workflow Memory** (Wang et al., 2024) | Experience abstraction | Distills past trajectories into reusable workflow summaries and procedural knowledge                 |
| **[[ACON]]** (Kang et al., 2025)                        | Context compression    | Optimizes context compression for long-horizon LLM agents; learns what to keep in the context window |

### 7.1.2. Benchmarks

| Benchmark                                  | Domain                 | Metrics                                                                                         | What it tests                                                                                                |
| ------------------------------------------ | ---------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Mind2Web** (Cross-Task split)            | Web navigation         | Element Accuracy (Ele Acc), Action F1, Step Success Rate (Step SR), Task Success Rate (Task SR) | Whether the agent can navigate real websites across multiple steps, grounding actions to correct UI elements |
| **[[AppWorld]]**                           | API interactions       | Success Rate — Easy / Medium / Hard / Average                                                   | Whether the agent can complete multi-step API workflows; "Hard" tasks require deep historical backtracking   |
| **[[NIAH]] (from RULER)**                  | Long-context retrieval | Compression Ratio, Recall@1                                                                     | Raw retrieval fidelity as context length scales from 4k to 32k tokens                                        |
| **Experience Retrieval Evaluation Subset** | Mind2Web-derived       | Recall@1, Recall@5, Recall@10, MRR                                                              | Isolated retrieval quality, independent of downstream task performance                                       |
| **SoM Ablation**                           | Mind2Web               | Ele Acc, Step SR, Latency                                                                       | Tests how much Set-of-Mark visual grounding contributes vs. text generation or bounding-box alternatives     |
| **Token Budget Stress Test**               | Mind2Web               | All Mind2Web metrics at 1024/2048/4096/8192 tokens                                              | Robustness under increasingly tight context windows                                                          |

### 7.1.3. Notable Results
On Mind2Web, OCR-Memory outperforms all baselines across every metric:

| Method         | Ele Acc   | Action F1 | Step SR   | Task SR  |
| -------------- | --------- | --------- | --------- | -------- |
| Zero-Shot      | 40.1%     | 46.2%     | 37.9%     | 2.2%     |
| Text-RAG       | 41.3%     | 48.2%     | 38.9%     | 2.7%     |
| MemoryBank     | 43.8%     | 49.5%     | 39.2%     | 3.3%     |
| AWM            | 49.1%     | 55.7%     | 42.6%     | 4.3%     |
| ACON           | 48.2%     | 54.1%     | 41.4%     | 4.1%     |
| **OCR-Memory** | **53.8%** | **59.2%** | **46.1%** | **4.8%** |
The clearest summary of what OCR-Memory trades and gains:

| -                             | Text-RAG | OCR-Memory |
| ----------------------------- | -------- | ---------- |
| Disk / Episode                | 18 KB    | 1.47 MB    |
| Text Tokens / Step into GPT-4 | 3,980    | 596        |
| Retrieval Latency / Step      | 0.3s     | 1.7s       |

Retrieval Quality:

| Method         | Recall@1  | Recall@5  | Recall@10 | MRR      |
| -------------- | --------- | --------- | --------- | -------- |
| **OCR-Memory** | **78.6%** | **93.4%** | **96.2%** | **0.84** |

---

# 8. Highlights
> [!PDF|255, 208, 0] [[OCR-Memory.pdf#page=1&annotation=631R|OCR-Memory, p.1]]
> > the finite context window of LLMs makes it impractical to store or revisit these high-fidelity trajectories in their entirety 
> 
> 

> [!PDF|] [[OCR-Memory.pdf#page=1&selection=112,48,116,38|OCR-Memory, p.1]]
> > This compromise often results in the loss of structural, temporal, or procedural details that are critical for complex downstream tasks such as debugging, error analysis, or multi-step planning.
> 
> Why summarisation dont works
