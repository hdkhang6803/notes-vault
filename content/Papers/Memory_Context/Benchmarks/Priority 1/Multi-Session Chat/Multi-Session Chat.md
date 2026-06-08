---
Tags:
Date: "2022"
Authors: Jing Xu, Arthur Szlam, Jason Weston (Facebook AI Research)
Venue: ACL
Paper: "Beyond Goldfish Memory∗ : Long-Term Open-Domain Conversation"
---
# 1. Terminology

| **Term**                     | Definition                                                                                                                                                                                                                                                       |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Multi-Session Chat (MSC)** | The benchmark dataset; human-human crowdworker conversations spanning up to 5 sequential sessions with time gaps between them                                                                                                                                    |
| **Session**                  | A single focused conversation episode of 6–7 turns per speaker, separated from the next by a simulated time gap                                                                                                                                                  |
| **Session Opening**          | The first utterance of a new session; re-engages the partner by referencing shared knowledge from prior sessions — treated as a distinct evaluation sub-task                                                                                                     |
| **Persona**                  | A fictional character profile (series of sentences on traits, events, opinions) assigned to crowdworkers to ensure diversity and mitigate privacy concerns                                                                                                       |
| **Extended Persona**         | The persona enriched across sessions via accumulated conversation summaries; captures evolving character depth                                                                                                                                                   |
| **Conversation Summary**     | Crowdworker-annotated compressed representation of salient personal facts from a past session, used as context for the next session                                                                                                                              |
| **Gold Summary**             | Human-annotated ground-truth summary used as oracle context during training/evaluation                                                                                                                                                                           |
| **Predicted Summary**        | Model-generated summary of past dialogue used as a proxy for gold summaries at inference time                                                                                                                                                                    |
| **no_summary label**         | Training label indicating a given dialogue turn contains no new information worth summarizing                                                                                                                                                                    |
| **Sparsity**                 | Fraction of turns for which a non-null summary is generated; controlled by subsampling the no_summary class during training                                                                                                                                      |
| **DPR**                      | Dense Passage Retrieval; a Transformer bi-encoder pre-trained on QA pairs, used for scoring and retrieving memory documents. Similarity is calculated between query and ALL documents' vectors (Contrast to only approximately similar vector using FAISS index) |
| **FiD**                      | Fusion-in-Decoder; encodes each of the top-N retrieved documents separately, then concatenates all encodings for the decoder to attend over                                                                                                                      |
| **FiD-RAG**                  | FiD variant using a RAG-trained retriever instead of standard pre-trained DPR                                                                                                                                                                                    |
| **SumMem**                   | Summary Memory; the proposed architecture that stores abstractive summaries (rather than raw dialogue) in long-term memory, retrieved via RAG/FiD/FiD-RAG                                                                                                        |
| **BST 2.7B**                 | BlenderBot pre-trained model (2.7B parameters) used as the backbone for all fine-tuning experiments                                                                                                                                                              |
| **Truncation**               | Hard cutoff on encoder input tokens (128, 512, or 1024) due to the fixed-length self-attention of standard Transformers                                                                                                                                          |
| **HIT**                      | Human Intelligence Task; unit of work on Amazon Mechanical Turk used to recruit and compensate crowdworkers                                                                                                                                                      |
| **Bi-encoders**              | Encode 2 texts separately then compare using cosine similarity                                                                                                                                                                                                   |
| **Cross-encoders**           | Concat 2 texts and pass them to the model at once so that tokens can interact with each other.                                                                                                                                                                   |

---

# 2. Metadata

| Property                     | Details                                                                                                                                       |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Total utterances**         | ~236K train / ~31K valid / ~30K test                                                                                                          |
| **Total summaries**          | 133,290 train / 25,459 valid / 24,375 test                                                                                                    |
| **Episodes (train)**         | 8,939 (Session 1 / PersonaChat); 4,000 (Sessions 1–3); 1,001 (Sessions 1–4)                                                                   |
| **Episodes (valid/test)**    | 500–501 per session, extending to Session 5                                                                                                   |
| **Avg utterances/episode**   | ~40.4 (1–3 sessions); ~53.3 (1–4 sessions); ~66 (valid/test, 5 sessions)                                                                      |
| **Avg utterance length**     | 21.4–23.0 tokens (BPE); avg full context ~1,614 tokens                                                                                        |
| **Vocabulary**               | ~37,366 unique tokens (MSC 1–3); ~23,387 (MSC 1–4)                                                                                            |
| **Sessions per episode**     | 1–5 (train up to 4; valid/test up to 5)                                                                                                       |
| **Turns per session**        | 12–14 total (6–7 per speaker)                                                                                                                 |
| **Personas**                 | 1,155 crowdsourced fictional profiles (disjoint across train/valid/test)                                                                      |
| **Annotators**               | 1,000+ English-speaking US crowdworkers via Amazon Mechanical Turk over ~6 months                                                             |
| **Train/valid/test overlap** | <0.26% utterance overlap across splits                                                                                                        |
| **Input format**             | Persona pair + prior session transcripts or summaries + current session dialogue history (text)                                               |
| **Output format**            | Next utterance (free-form text); or summary sentence / no_summary label (for the summarization sub-task)                                      |
| **Annotation format**        | Per-turn summary labels (text sentence or `no_summary`); session-level engagingness ratings (1–5); per-turn binary topic-reference attributes |

---

# 3. What it Measures (What)

## 3.1. Task Definition

The task is framed as next-utterance generation conditioned on the current session context plus some representation of all prior sessions (raw dialogue, gold summary, or predicted summary).

## 3.2. Example Records

Each record consists of:

- **Persona pair:** two fictional character profiles (e.g., _"I enjoy bow hunting. My favourite food is jerk chicken with red beans and rice."_)
- **Session history:** one or more prior session transcripts or summaries, labeled with a simulated time gap (e.g., _[6 days later]_)
- **Current session dialogue:** a sequence of alternating utterances to continue
- **Per-turn summary annotations:** a factual summary sentence capturing new salient information (e.g., _"The Redskins and the Packers are my favorite football teams. I played football in college."_) or the `no_summary` label
- **Session opening target:** the ground-truth first utterance of a new session, referencing known shared history (e.g., _"How is your robot doing? Have you had time to play any video games with it yet?"_)

### 3.2.1. Example 1: Summary Annotations (Figure 1)

This shows the **turn-level summary annotation task*: what the crowdworker labeled vs. what the model predicted.

| Utterance (Speaker 2)                                  | Gold Label                                                                                 | Model Prediction                                                                           |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| "Redskins and packers. I played college."              | The Redskins and the Packers are my favorite football teams. I played football in college. | I played football in college. My favorite football teams are the Redskins and the Packers. |
| "Yes, I've. I picked up a lovely yellow blouse there." | I have been to Spain. I bought a yellow blouse in Spain.                                   | I have travelled to Spain.                                                                 |
| "Okay, so tell me more about yourself"                 | NONE                                                                                       | NONE                                                                                       |
### 3.2.2. Example 2: Sessions conversation

**Persona assigned to Speaker 1:** plays folk music, hopes to be a professional musician, can play guitar and piano, likes to sing, not a big drinker, works as a custodian, loves listening to classic/baroque music.

**Persona assigned to Speaker 2:** interested in how instruments are made, curious about forensics and unsolved crimes, rides bike as favourite activity, works in construction like his dad, prefers lemonade/coconut water.

Session 1

> **S2:** Hello, how are you doing?
> 
> **S1:** Hello. Pretty good, thanks. And yourself?
> 
> **S2:** Awesome, I just got back from a bike ride.
> 
> **S1:** Cool! Do you spend a lot of time biking?
> 
> ...
> 
> **S1:** Exactly. Do you have other hobbies besides biking?

Session 2: _[6 hours later]_

> **S2:** Today's weather was beautiful for biking. It was the first thing I did after my long day working my construction job. Did you get to play any guitar or piano today? Or perhaps a drink at a local bar?
> 
	....
> **S2:** Since you're so interested in music, do you ever get out to see local bands? I used to go to concerts every few months when I was younger.
### 3.2.3. Example 3: Session Opening Predictions (Figure 4)

This shows the **session opening sub-task**: the model must generate the first message of a new session using gold summaries as context.

| Time Gap         | Gold Label (human)                                  | Model Prediction                                                                   |
| ---------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------- |
| _[3 days later]_ | My cow is doing very well and eating well now.      | How is your robot doing? Have you had time to play any video games with it yet?    |
| _[5 days later]_ | I love my new outfits will you help me shop again?  | Have you dyed your hair since we last spoke? I'm thinking about dying mine purple. |
| _[4 days later]_ | My interview went well. I think I will get the job. | How is your garden coming along? Have you planted any new vegetables or flowers?   |
## 3.3. Metrics

| Metric                        | Type      | Description                                                                                                                                                                                                                                                                                                                                                                                                         |
| ----------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Perplexity**                | Automatic | Primary metric; computed per-session and specifically on session openings; lower is better<br><br>![[Pasted image 20260518144235.png]]<br><br>At each token position, the model assigns a probability to every word in the vocabulary. If the model assigns high probability to the word the human actually wrote, perplexity stays low. If the model is surprised (assigns low probability), perplexity shoots up. |
| **BLEU-1/2/4**                | Automatic | N-gram overlap between generated and reference responses<br><br>![[Pasted image 20260518144351.png]]                                                                                                                                                                                                                                                                                                                |
| **ROUGE-1/2/L**               | Automatic | Token-level overlap between generated and reference responses<br><br>![[Pasted image 20260518144454.png]]                                                                                                                                                                                                                                                                                                           |
| **Engaging Response Rate**    | Human     | % of turns rated as engaging by crowdworker conversational partners                                                                                                                                                                                                                                                                                                                                                 |
| **Final Engagingness Rating** | Human     | Overall conversation quality score out of 5 collected at end of each session                                                                                                                                                                                                                                                                                                                                        |
| **Reference own topic %**     | Human     | % of turns where the model correctly references its own prior persona/topics                                                                                                                                                                                                                                                                                                                                        |
| **Reference other's topic %** | Human     | % of turns where the model correctly references the partner's prior persona/topics                                                                                                                                                                                                                                                                                                                                  |
| **Sparsity**                  | Automatic | For summary models: fraction of turns that produce a non-null summary                                                                                                                                                                                                                                                                                                                                               |

---

# 4. Uniqueness (Why)

## 4.1. Other Benchmarks

| Dataset                  | Episodes  | Avg Utts/Episode | Sessions/Episode |
| ------------------------ | --------- | ---------------- | ---------------- |
| [[Pushshift.io Reddit]]  | —         | 3.2              | 1                |
| [[PersonaChat]]          | 8,939     | 14.7             | 1                |
| [[Wizard of Wikipedia]]  | 18,430    | 9.0              | 1                |
| [[Daily Dialog]]         | 22,236    | 3.9              | 1                |
| [[Empathetic Dialogues]] | 24,850    | 2.6              | 1                |
| **MSC with 3 sessions**  | **4,000** | **40.4**         | **3**            |
| **MSC with 4 sessions**  | **1,001** | **53.3**         | **4**            |
All prior open-domain dialogue benchmarks consist of a single session with 2–15 turns. None capture temporal re-engagement dynamics across multiple sessions separated by real (simulated) time gaps.

## 4.2. Uniqueness
- **First public multi-session open-domain dialogue benchmark with Temporal re-engagement**: no comparable public resource existed at publication time; prior datasets were all single-session.
- **Session opening as a distinct evaluation target**: the first utterance of each new session is isolated as a structurally special, memory-demanding utterance type and evaluated separately, revealing ~2 perplexity point gaps invisible to standard mid-session evaluation.
- **Paired human-written summary annotations**: every session boundary has crowdworker-written summaries of salient personal facts (both self and partner), enabling supervised training of a summarization memory module rather than only retrieval from raw logs.
- **Explore the extended persona**: personas deepen organically through conversation rather than being fixed pre-conversation inputs, enabling study of character depth over time.
---

# 5. Generation Method (How)

**Session 1** is drawn directly from the PersonaChat dataset (Zhang et al., 2018) — short first-meeting conversations between speakers playing assigned fictional personas from a pool of 1,155 crowdsourced character profiles. Train/valid/test use strictly disjoint persona sets.

**Sessions 2–5** were collected via Amazon Mechanical Turk over 6 months with the following key design decisions:
- **Role-playing constraint:** crowd workers play assigned fictional personas rather than speaking as themselves.
- **Simulated time gaps:** a random elapsed time (1–7 hours or 1–7 days) is displayed before each new session.
- **Worker non-continuity by design:** the pair of workers playing a given persona pair in session N may differ from those in session N+1; only the character roles and accumulated conversation history are held constant. 
- **Summary collection as a separate HIT:** after each session, a separate worker pool reads all prior dialogues and writes compressed bullet-point summaries of important personal facts for both speakers. These will be used to train the summarizer or used for subsequent sessions' context.
- **Quality control pipeline:** onboarding tasks with high pass thresholds, per-turn bad-behavior flags by the conversation partner, final engagingness ratings per conversation, and automatic filters on minimum message length and low-quality-turn rate. Poor-performing workers are blocked from future HITs and low-quality dialogues are filtered from the final release.
- **Data quality verification:** utterance uniqueness is measured within and across splits; overlap rates are <0.26%, confirming negligible leakage between train and test.

---

# 6. Gaps

1. **Synthetic time gaps vs. real temporal drift:** simulated hours/days cannot capture genuine forgetting, language evolution, or mood changes that occur in real longitudinal conversations
2. **Perplexity as proxy for engagement:** automatic metrics (perplexity, BLEU, F1) correlate only loosely with human engagingness ratings; the human evaluation is small-scale and limited to session 5, making it difficult to draw statistically robust conclusions across all sessions.
3. **No multimodal or non-text grounding:** all memory is purely textual; real long-term human conversations involve shared images, links, and references to external events that the benchmark cannot capture or evaluate.

---

# 7. Highlights
> [!PDF|255, 208, 0] [[Multi-Session Chat.pdf#page=1&annotation=1067R|Multi-Session Chat, p.5180]]
> > a major aspect missing from the current state of the art is that human conversations can take place over long time frames, whereas the currently used systems suffer in this setting.

> [!PDF|255, 208, 0] [[Multi-Session Chat.pdf#page=1&annotation=1073R|Multi-Session Chat, p.5180]]
> > we collect and release1 a new English dataset, entitled Multi-Session Chat (MSC) that consists of human-human crowdworker chats over 5 sessions, with each session consisting of up to 14 utterances, where the conversationalists reengage after a number of hours or days and continue chatting. 

> [!PDF|255, 208, 0] [[Multi-Session Chat.pdf#page=1&annotation=1076R|Multi-Session Chat, p.5180]]
> > When reengaging, conversationalists often address existing knowledge about their partner to continue the conversation in a way that focuses and deepens the discussions on their known shared interests, or explores new ones given what they already know.

