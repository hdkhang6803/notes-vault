---
Tags:
Date: "2021"
Authors: Yonatan Bisk, Xingdi Yuan, Adam Trischler, Marc-Alexandre Côté, Matthew Hausknecht
Venue: ICLR
Paper: "ALFWORLD: ALIGNING TEXT AND EMBODIED ENVIRONMENTS FOR INTERACTIVE LEARNING"
---
# 1. Terminology

| Term                                               | Definition                                                                                                                                                                                                                                                                  |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ALFWorld**                                       | The main framework introduced in this paper. It aligns two parallel worlds: a text-based game environment (TextWorld) and a visually rendered, physics-based household simulator (ALFRED), so agents can train in one and transfer to the other.                            |
| **ALFRED**                                         | (_Action Learning From Realistic Environments and Directives_) A pre-existing large-scale benchmark for vision-language instruction following in embodied environments, built on the AI2-THOR simulator. ALFWorld uses ALFRED's tasks and scenes as its embodied component. |
| **TextWorld**                                      | A text-based game engine that programmatically generates interactive text environments. In ALFWorld, it serves as the abstract, language-only training ground — the "text twin" of the ALFRED scenes.                                                                       |
| **BUTLER**                                         | (_Building Understanding in Textworld via Language for Embodied Reasoning_) The new agent introduced alongside ALFWorld, consisting of three modular components: BRAIN (text planner), VISION (visual-to-text state estimator), and BODY (low-level action controller).     |
| **Embodied Agent**                                 | An AI agent that perceives and interacts with a simulated physical environment through visual observations (camera images) and low-level physical actions (move, rotate, pick up, etc.).                                                                                    |
| **PDDL**                                           | (_Planning Domain Definition Language_) A formal symbolic language used to describe world states and valid actions. ALFWorld uses PDDL internally to represent each ALFRED scene and generate the equivalent TextWorld game.                                                |
| **Imitation Learning (IL)**                        | A training paradigm where an agent learns by imitating expert demonstrations, rather than learning from trial-and-error rewards. BUTLER is trained with IL using a rule-based expert.                                                                                       |
| **DAgger**                                         | (_Dataset Aggregation_) An interactive imitation learning algorithm where the agent alternately takes actions and queries the expert for corrections. This allows the agent to recover from its own mistakes, unlike static supervised learning.                            |
| **Goal-condition Success Rate**                    | A partial-credit metric from ALFRED. A task is broken into sub-goals (e.g., heat object, place object). The score is the fraction of sub-goals completed, even if the full task fails. Reported in parentheses alongside task success rate.                                 |
| **Mask R-CNN**                                     | A computer vision model for instance segmentation; BUTLER::VISION uses Mask R-CNN to translate visual frames into text descriptions.                                                                                                                                        |
| **Beam Search**                                    | A transformer decoding strategy that explores multiple candidate token sequences simultaneously and picks the most likely sequence after all of them have been produced.                                                                                                    |
| **A* Planner**                                     | A classical pathfinding algorithm. BUTLER::BODY uses A* on a pre-built grid map of the scene to navigate between receptacle locations via the shortest path.                                                                                                                |
| **Receptacle**                                     | Any container or surface where objects can be placed (e.g., fridge, microwave, countertop, drawer). Receptacles are static fixtures in the scene; objects are portable.                                                                                                     |
| **High-level vs. Low-level Actions**               | High-level: abstract, language-described actions like `clean cloth with sinkbasin` (used in TextWorld). Low-level: primitive physical commands like `MOVEAHEAD`, `ROTATELEFT`, `PICKUP` (used in the embodied ALFRED environment).                                          |
| **Domain Gap**                                     | The mismatch between two environments that makes policies learned in one environment not directly applicable to the other. Example: TextWorld allows any object in any receptacle; the embodied world enforces physical size constraints.                                   |
| **Episode**                                        | A complete run through of a task from start to finish                                                                                                                                                                                                                       |
| **Wall-clock efficiency** (or **wall-clock time**) | refers to the actual, real-world time that passes                                                                                                                                                                                                                           |
| Fast Downward                                      | A tool that automatically figures out a sequence of actions to achieve a goal, given a symbolic description of the world state and valid actions.                                                                                                                           |

---

# 2. Metadata

| Property                     | Value                                                                                                                                    |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Simulator Base**           | AI2-THOR (embodied) + TextWorld engine (text)                                                                                            |
| **Task Types**               | 6 (Pick & Place, Examine in Light, Clean & Place, Heat & Place, Cool & Place, Pick Two & Place)                                          |
| **Total Training Tasks**     | 3,553                                                                                                                                    |
| **Seen Evaluation Tasks**    | 140 (across all 6 task types)                                                                                                            |
| **Unseen Evaluation Tasks**  | 134 (across all 6 task types)                                                                                                            |
| **Room Environments**        | 120 rooms: 30 kitchens, 30 bedrooms, 30 bathrooms, 30 living rooms                                                                       |
| **Object Classes**           | 73 object classes, each with 1–10 instances per scene                                                                                    |
| **Detector Training Images** | 50,000 images (from ALFRED expert replays, balanced across room types)                                                                   |
| **Max Steps per Episode**    | 50 steps (TextWorld); forced termination at limit                                                                                        |
| **Training Episodes**        | 100,000 (BUTLER::BRAIN in TextWorld); 50,000 for strategy comparisons                                                                    |
| **Observation Queue Length** | 5 most recent unique observations (default)                                                                                              |
| **Action Space (TextWorld)** | 11 high-level text actions: `goto`, `take`, `put`, `open`, `close`, `toggle`, `clean`, `heat`, `cool`, `inventory`, `examine`            |
| **Action Space (Embodied)**  | 9 low-level primitives: `MOVEAHEAD`, `ROTATELEFT`, `ROTATERIGHT`, `LOOKUP`, `LOOKDOWN`, `PICKUP`, `PUT`, `OPEN`, `CLOSE`, `TOGGLEON/OFF` |
| **Language Modality**        | Template-based goals (training); human-annotated goals (evaluation)                                                                      |
| **Human Goal Vocabulary**    | 66 unseen verbs, 189 unseen nouns (vs. training templates)                                                                               |
| **Training Speed Advantage** | TextWorld is ~7× faster than embodied training (CPU-only vs. GPU rendering + physics)                                                    |
Size Breakdown by Task Type

| Task Type        | Train     | Seen Eval | Unseen Eval |
| ---------------- | --------- | --------- | ----------- |
| Pick & Place     | 790       | 35        | 24          |
| Examine in Light | 308       | 13        | 18          |
| Clean & Place    | 650       | 27        | 31          |
| Heat & Place     | 459       | 16        | 23          |
| Cool & Place     | 533       | 25        | 21          |
| Pick Two & Place | 813       | 24        | 17          |
| **Total**        | **3,553** | **140**   | **134**     |

---

# 3. What It Measures (What)

## 3.1. Task Definition

ALFWorld measures an agent's ability to **complete multi-step household tasks** by combining abstract planning (expressed in natural language) with concrete physical execution (in a visually rendered environment). Each task is described only by a **goal description** and the agent must figure out the steps on its own, without step-by-step instructions.

|Task Type|Goal Example|Key Challenge|
|---|---|---|
|**Pick & Place**|_"Put a plate on the coffee table"_|Navigation + search + placement|
|**Examine in Light**|_"Look at the alarm clock under the lamp"_|Find both object and light source, toggle lamp|
|**Clean & Place**|_"Clean the knife and put it in the drawer"_|Find sink, wash object, then place|
|**Heat & Place**|_"Put a hot mug on the counter"_|Find microwave, heat object, then place|
|**Cool & Place**|_"Put a cool bottle on the shelf"_|Find fridge, cool object, then place|
|**Pick Two & Place**|_"Put two pencils in the drawer"_|Two search-and-place cycles for matching objects|

> **Example walkthrough (Clean & Place in TextWorld):**
> 
> Task: _"Put a clean cloth in bathtubbasin"_
> 
> 1. `go to countertop 1` → finds cloth 1 and cloth 2 on the countertop
> 2. `take cloth 1 from countertop 1`
> 3. `go to sinkbasin 1`
> 4. `clean cloth 1 with sinkbasin 1` → cloth is now clean
> 5. `go to bathtubbasin 1`
> 6. `put cloth 1 in/on bathtubbasin 1` → **You won!**
>  
> The same task in the **embodied ALFRED environment** requires the agent to navigate via low-level actions (`MOVEAHEAD`, `ROTATELEFT`, etc.), identify the cloth using Mask R-CNN, and physically execute the placement (adding perception and motor control challenges on top of the planning problem).

ALFWorld distinguishes between two evaluation conditions:

- **Seen:** Same room types as training, but object positions, quantities, and visual appearances differ (e.g., two blue pencils on a shelf instead of three red pencils in a drawer). Measures in-distribution generalization.
- **Unseen:** Entirely new room layouts with different receptacle placements never seen during training. Measures out-of-distribution generalization
## 3.2. Example records:

--- start-multi-column: ID_pdf2
```column-settings
Number of Columns: 2
Largest Column: standard
```

<video src="Attachment/ALFWorld.mp4" controls ></video>


--- column-break ---

You are in the middle of a room. Looking quickly around you, you see a drawer 2, a shelf 5, a drawer 1, a shelf 4, a sidetable 1, a drawer 5, a shelf 6, a shelf 1, a shelf 9, a cabinet 2, a sofa 1, a cabinet 1, a shelf 3, a cabinet 3, a drawer 3, a shelf 11, a shelf 2, a shelf 10, a dresser 1, a shelf 12, a garbagecan 1, a armchair 1, a cabinet 4, a shelf 7, a shelf 8, a safe 1, and a drawer 4.   
  
Your task is to: _put some vase in safe._   
  
**> go to shelf 6  
**You arrive at loc 4. On the shelf 6, you see a vase 2.  
  
**> take vase 2 from shelf 6  
**You pick up the vase 2 from the shelf 6.  
  
**> go to safe 1  
**You arrive at loc 3. The safe 1 is closed.  
  
**> open safe 1  
**You open the safe 1. The safe 1 is open. In it, you see a keychain 3.  
  
**> put vase 2 in/on safe 1  
**You won!

--- end-multi-column

## 3.3. Metrics

ALFWorld reports two complementary metrics, each broken down by seen/unseen split:

- **Task Success Rate (%)**: A task is considered successful only if the final world state satisfies all goal conditions (e.g., the object is clean AND in the correct receptacle). 
- **Goal-condition Success Rate (%)**: Each task is decomposed into atomic sub-goals (e.g., for _"heat a potato and put it on the counter"_: (1) heat any object, (2) put a potato on the counter, (3) heat a potato and put it on the counter). The score is the proportion of sub-goals satisfied, even if the final task fails.

> **Example of goal-condition scoring:** Task: _"Put a hot potato on the countertop"_ → 3 sub-goals
> 
> - Agent heats the potato ✓ (1/3 = 0.33)
> - Agent fails to place it on the countertop ✗
> - Goal-condition score = 0.33; Task success = 0

- **Training speed (episodes/second):** Measures wall-clock efficiency of each training strategy. TextWorld achieves ~6.1 eps/s vs ~0.9 eps/s for embodied-only training — a 7× advantage.

---

# 4. Uniqueness (Why)
## 4.1. Other benchmarks
- [[TextWorld]]
- [[Jericho]]
- [[ALFRED]] 
- [[MAttNet]]
- [[ViLBERT]]
- [[BabyAI]]

## 4.2. Uniqueness

ALFWorld fills a gap that no prior benchmark addressed: the simultaneous need for **interactive language learning** and **embodied execution** in a **paired, aligned environment**. 
=> Evaluate models on both textual ability and physical navigation ability

---
# 5. Generation Method (How)

ALFWorld is constructed by connecting TextWorld and ALFRED through a shared symbolic state representation. The generation pipeline has four main stages:

### 5.1.1. Stage 1: Shared Symbolic State via PDDL

Every ALFRED scene is described using **PDDL (Planning Domain Definition Language)**, which encodes world state as a set of logical predicates:

```
s_t = at(player, microwave) ⊗ in(mug, microwave) ⊗ closed(microwave) ⊗ openable(microwave)
```

Actions modify the PDDL state (e.g., `open microwave` replaces `closed(microwave)` with `open(microwave)`)
### 5.1.2. Stage 2: TextWorld Game Construction

The **TextWorld Engine** takes a PDDL scene description and automatically generates a playable text game in two sub-steps:
- **Planner (Fast Downward):** A domain-independent classical planning system that maintains and updates the PDDL state as the agent takes actions, enforcing valid transitions.
- **Text Generator:** Use a context-sensitive grammar designed specifically for ALFRED environments. It samples text templates conditioned on the current state and last action to produce natural-sounding observations (e.g., `"You arrive at loc 25. On the countertop 1, you see a cloth 2, a soapbottle 1, and a cloth 1"`).

> **Example observation template for `goto`:**
> (a) "You arrive at {loc_id}. On the {recep_id}, you see a {obj1_id}, ... and a {objN_id}."
> (b) "You arrive at {loc_id}. The {recep_id} is closed."
### 5.1.3. Stage 3: Goal Description Generation

Goal descriptions are generated in two ways:
- **Templated Goals** (used for training): For each task type, two templates are defined and sampled with equal probability, e.g.:
	- Pick & Place → `"put a {obj} in {recep}"` / `"put some {obj} on {recep}"`
	- Heat & Place → `"put a hot {obj} in {recep}"` / `"heat some {obj} and put it in {recep}"`
- **Human-Annotated Goals** (used for evaluation): Drawn from the original ALFRED crowdsourced annotations, containing **66 unseen verbs** and **189 unseen nouns** relative to the templated training vocabulary.

### 5.1.4. Stage 3: Dataset Splits Construction
For each of the 6 task types, three splits are constructed:

| Split         | Description                                                                                                                 |
| ------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Train**     | Full training set (e.g., 790 Pick & Place tasks)                                                                            |
| **Seen Eval** | Known task instances in rooms seen during training, but with different object locations, quantities, and visual appearances |
| **Unseen**    | New task instances in entirely unseen rooms with different receptacles and scene layouts                                    |
The seen/unseen distinction is deliberate: **seen** tests in-distribution generalization; **unseen** tests out-of-distribution generalization.

---

## 5.2. Contributions

- **ALFWorld Environment:** The first parallel, interactive, aligned text-and-embodied benchmark

- **BUTLER Agent:** A modular, upgradeable cross-modal agent architecture which cleanly separates the sub-problems of abstract planning, visual scene understanding, and low-level motor control. Each component can be independently improved

- Empirical proof that text pre-training beats embodied-only and hybrid training for generalization

- Zero-shot cross-modal generalization

---

## 5.3. Gaps
- **Persistent Domain Gap Between TextWorld and the Embodied World:** This gap arises from physical constraints that TextWorld ignores (e.g., you cannot put a large pot in a small microwave). 
	=> Incorporate domain randomization, physics-aware PDDL constraints, or a small amount of embodied fine-tuning to close this gap.

- **Very Low Absolute Performance**: Even BUTLER-ORACLE with perfect perception and teleportation achieves only 37% (seen) and 26% (unseen). These numbers suggest the benchmark is far from solved
- **Limited Task Coverage and Domain Scope:**  ALFWorld covers only 6 household task types and is constrained to indoor household scenarios. There is no evaluation of task composition, adversarial object placements, or multi-agent settings.
	=> Extend ALFWorld to include compositional tasks, more diverse environments (offices, outdoor spaces), and multi-agent collaboration scenarios.

# 6. Highlights
> [!PDF|255, 208, 0] [[ALFWORLD ALIGNING TEXT AND EMBODIED.pdf#page=1&annotation=895R|ALFWORLD ALIGNING TEXT AND EMBODIED, p.1]]
> > We hypothesize that, learning to solve tasks using abstract language, unconstrained by the particulars of the physical world, enables agents to complete embodied tasks in novel environments by leveraging the kinds of semantic priors that are exposed by abstraction and interaction.

> [!PDF|255, 208, 0] [[ALFWORLD ALIGNING TEXT AND EMBODIED.pdf#page=5&annotation=910R|ALFWORLD ALIGNING TEXT AND EMBODIED, p.5]]
> > We note that a given pre-built grid-map of receptacle locations is a strong prior assumption, but future work could incorporate existing models from the vision-language navigation literature (Anderson et al., 2018a; Wang et al., 2019) for map-free navigation.

> [!PDF|255, 208, 0] [[ALFWORLD ALIGNING TEXT AND EMBODIED.pdf#page=6&annotation=913R|ALFWORLD ALIGNING TEXT AND EMBODIED, p.6]]
> > transferring between modalities involves overcoming domain gaps that are present in the real world but not in TextWorld. For example, the physical size of objects and receptacles must be respected – while TextWorld will allow certain objects to be placed inside any receptacle, in the embodied world it might be impossible to put a larger object into a small receptacle (e.g. a large pot into a microwave).
> 
> 