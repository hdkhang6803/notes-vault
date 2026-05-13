---
Tags:
Date: "2026"
Authors: Yiming Xiong, Shengran Hu, Jeff Clune (University of British Columbia / Vector Institute)
Venue: ICLR 2026
Paper: Learning to Continually Learn via Meta-learning Agentic Memory Designs
---
# 1. Terminology

| **Term**                                | **Plain Explanation**                                                                                                                                                        |
| :-------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Foundation Model (FM)**               | A large pre-trained model (e.g. GPT-4) used as the backbone of an agent. It is stateless: it has no memory across independent calls.                                         |
| **Continual Learning**                  | The ability to accumulate knowledge from a sequence of tasks and improve over time, without forgetting earlier experiences.                                                  |
| **Memory Module**                       | A component attached to an agent that stores past interactions (trajectories, insights, strategies) and retrieves relevant ones for new tasks.                               |
| **Token-level Memory**                  | A memory type where information is stored as text and injected into the FM's prompt at inference time. No model weights are changed.                                         |
| **Parametric Memory**                   | Memory stored by fine-tuning model weights. Slower and less interpretable.                                                                                                   |
| **Meta-learning ("Learning to Learn")** | A paradigm where a higher-level process learns how to learn, rather than learning a specific task. Here, ALMA learns how to design memory rather than designing it manually. |
| **Open-ended Exploration**              | A search strategy that explicitly values novelty and diversity, rather than greedily optimizing for the single best candidate. Inspired by evolutionary algorithms.          |
| **Memory Collection Phase**             | The initial phase where an agent runs on tasks without using memory, and the resulting trajectories are stored to build a memory state.                                      |
| **Deployment Phase**                    | The phase where the agent uses the built memory to solve new tasks, with optional dynamic updates.                                                                           |
| **Trajectory**                          | A full sequence of states and actions the agent takes to complete (or fail at) a task.                                                                                       |
| **Meta Agent**                          | An FM-powered agent (powered by GPT-5 in this work) that generates, critiques, and modifies candidate memory designs in code.                                                |
| **Memory Design Archive**               | A growing database of all previously explored memory designs and their evaluation results, used to guide future search steps.                                                |
| **Sampling Score**                      | A formula combining a design's success rate and how often it has been sampled, used to balance exploration (trying new designs) and exploitation (refining good ones).       |
| **Static Mode**                         | Deployment setting where the memory is built once and stays fixed throughout evaluation. Used to measure how well the memory captures transferable knowledge.                |
| **Dynamic Mode**                        | Deployment setting where memory is updated as new task results come in. Tests adaptation under distribution shift.                                                           |

# 2. Paper Summary (What)
**ALMA** (**A**utomated meta-**L**earning of **M**emory designs for **A**gentic systems) proposes to automate memory design process. 
Instead of human designing how the agent stores and retrieves experience, a _Meta Agent_ searches over the space of possible memory designs (expressed as Python code) using an open-ended evolutionary loop. 
The best-found memory design is then deployed in the target agent system.
# 3. What it solves (Why)
Agentic System needs memory **BUT** memory design is handcrafted by human => Brittle, Heuristics, Experience-based, Time-consuming
==> Let the machine decides itself --> ALMA
# 4. Methodology (How)
ALMA's core loop has four stages, repeated for a fixed number of iterations (11 steps in experiments, producing 43 candidate designs):
## 4.1. Stage 1: Sample from the Archive
- The Meta Agent samples up to 5 previously explored memory designs from the archive. 
- Designs with higher success rates but fewer prior samples are favored.
- All designs retain a non-zero sampling probability, keeping the search open.

> **Example:** In iteration 4, the archive contains 12 designs. A design with 25% success rate that was only tried once gets a higher sampling score than one with 30% success tried five times, so it is sampled to guide the next candidate.
## 4.2. Stage 2: Ideate & Plan

- Given a sampled design, its source code, and logged interaction trajectories (some successful, some failed), the Meta Agent reflects on what is working and what is not. 
- It produces a structured plan for a new design, identifying which memory sub-modules to change, add, or remove.

> **Example (Baba Is AI):** The Meta Agent notices that retrieved memory make the agent loops in failure patterns. It proposes adding a Property Validation sub-module that filters rules that are provably blocked, plus a Strategy Library that stores explicit rule-manipulation strategies. These become stepping stones toward the final design.

## 4.3. Stage 3: Implement & Debug
- The Meta Agent writes the full memory design as a Python class, inheriting from the abstract MemoStructure template. 
- If the code crashes during a trial run (on 5 tasks), the Meta Agent self-reflects and debugs for up to 3 rounds.
## 4.4. Stage 4: Evaluate & Archive

- The design is evaluated on the full learning set:
	- Memory Collection Phase (agent runs without memory, trajectories are stored)
	- Deployment Phase (agent runs with retrieved memory, success rate recorded). 
- The result is archived, and the loop continues.
- After all iterations, the design with the best success rate is selected as the learned memory design.

## 4.5. The Two Interfaces

Every explored memory design must implement two methods:

- `general_update()` called after a task; extracts and stores knowledge from the interaction log.
- `general_retrieve()` called before a task; queries the memory and formats relevant knowledge to inject into the agent's prompt.
    
Internally, designs can compose multiple sub-modules (e.g., a graph database, an embedding-based retrieval store, a strategy summarizer) chained in any order

> **Example (MiniHack):** The best-learned design chains five sub-modules: 
> - TaskSchemaLayer (parses the map and current situation), 
> - SpatialPriorLayer (a knowledge graph of entity–action relations), 
> - StrategyLibraryLayer (distilled strategies from past episodes), 
> - RiskAndInteractionLayer (safety heuristics), 
> - ReflexRulesLayer (immediate next-move advice). 
> 
> The final output injected to the agent is a structured JSON with a plan outline, bullet-point tips, spatial priors, and an action mask.
# 5. Contributions
1. **Principled Automation of a Bottleneck**: Hand-crafting memory designs is time-consuming and requires domain expertise. ALMA automates this end-to-end. The code-based search space is Turing complete, meaning no type of memory is excluded a priori.
2. **Domain Specialization Without Human Effort:** ALMA discovers different memory architectures for different domains with no human steering.
3. **Strong Generalization:** Memory designs learned with a weak FM (GPT-5-nano) transfer robustly to a stronger FM (GPT-5-mini), showing the improvements are structural rather than model-specific.
4. **Open-ended Exploration is Key** : The ablation showing open-ended search better than greedy search is an important empirical contribution. 
5. **Interpretability**: Because memory designs are expressed as Python code with named sub-modules, they are human-readable and inspectable.
6. **Safety-Aware Execution**: Each generated memory design runs in an isolated sandbox. Human oversight of learned designs is performed to check for prompt injection or other adversarial behaviors
# 6. Gaps
1. **No Online (In-Context) Adaptation**: The current framework has a hard separation between a learning phase and a deployment phase.(computational cost, each evaluation requires many rollouts).

2. **High Computational Cost of Meta-Learning**: ALMA uses GPT-5 as the Meta Agent, and each learning step evaluates candidate designs by running full agentic rollouts. The total compute budget is substantial and not reported in the paper. 
	=> Cheaper meta-learning surrogates: predicting memory design performance from code features without full rollout, or using smaller code-generation models for implementation.

3. **Bounded by the Underlying FM**:  The memory designs are limited by what the underlying FM can implement correctly and what tools are exposed to the Meta Agent (Chroma DB, NetworkX graph, a small set of agent-as-tool wrappers).
	=> Expanding the search space to include novel database backends, custom compression schemes, or hybrid parametric-token designs, potentially with automated scaffolding.

4. **Evaluation Only on Text-Based Domains**: All four benchmarks operate in fully text-based, discrete-action environments. It is unclear whether the learned memory designs and the meta-learning process transfer to multimodal, continuous-control, or open-world settings.
	=> Evaluating ALMA in embodied, vision-language, or robotics domains where memory structure may need to encode perceptual or spatial information beyond text.

5. **Limited Safety Mechanism**: The paper acknowledges that AI-generating algorithms introduce safety risks (learned designs might encode unintended behaviors). The current mitigation is sandboxing and manual human review. **At scale, this approach does not hold**. 
	=>Research gap: Automated safety inspection of generated memory designs, e.g., property-based testing, invariant checking, or small-scale adversarial prompting, as a complement to human review.

6. **Sensitivity to the Meta Agent's Capability**: The paper does not ablate the Meta Agent's strength, so it is unclear how well ALMA performs with a weaker or open-source model. This is a reproducibility and accessibility concern.

	=> Research gap: Testing ALMA with open-weight Meta Agents (e.g., Llama, Mistral) to understand the minimum Meta Agent capability needed for useful memory design search.

7. **Only Token-Level Memory Explored:** By design, ALMA focuses exclusively on token-level (text-in-context) memory, excluding parametric and latent memory. 
	=> Research gap: Extending the "learning to design" paradigm to parametric and latent memory, perhaps in combination with lightweight fine-tuning or adapter methods.


# 7. Benchmarks

## 7.1. Other baselines:
- [[Trajectory Retrieval]]: store and retrieve similar trajectories
- [[Reasoning Bank]]: organize and extract experience on a per-task basis, enabling the retrieval of experience relevant to each task.
- [[Dynamic Cheatsheet]]: A global semantic memory incrementally accumulates experience by FMs across all past trajectories 
- [[G-Memory]]: A hierarchical graph-based memory design
## 7.2. Benchmarks:
- [[ALFWorld]]: text-based simulation of household tasks  
- [[TextWorld]]: text adventure games
- [[Baba Is AI]]: Puzzles where agent manipulate game rules
- [[MiniHack]]: long-horizon decision-making in dungeon environments

| Memory Design        | ALFWorld  | TextWorld | Baba Is AI | MiniHack  | Overall Avg |
| -------------------- | --------- | --------- | ---------- | --------- | ----------- |
| No Memory            | 2.9%      | 5.4%      | 9.5%       | 6.7%      | 6.1%        |
| Trajectory Retrieval | 5.2%      | 2.7%      | 19.0%      | 7.5%      | 8.6%        |
| ReasoningBank        | 5.2%      | 5.3%      | 9.5%       | 9.8%      | 7.5%        |
| Dynamic Cheatsheet   | 5.7%      | 4.3%      | 9.5%       | 9.2%      | 7.2%        |
| G-Memory             | 7.6%      | 2.1%      | 14.3%      | 6.8%      | 7.7%        |
| **ALMA**             | **12.4%** | **6.2%**  | **19.0%**  | **11.7%** | **12.3%**   |
# 8. Highlights:
> [!PDF|255, 255, 0] [[Learning to Continually Learn via Meta-learning Agentic Memory Designs.pdf#page=4&annotation=4874R|Learning to Continually Learn via Meta-learning Agentic Memory Designs, p.4]]
> > static mode assesses how effectively the agent leverages a fixed memory to solve new tasks,

> [!PDF|] [[Learning to Continually Learn via Meta-learning Agentic Memory Designs.pdf#page=4&selection=59,41,61,50|Learning to Continually Learn via Meta-learning Agentic Memory Designs, p.4]]
> > dynamic mode measures how well a memory design adapts to a new task distribution through dynamic updates and retrieval
> 
> 


> [!PDF|] [[Learning to Continually Learn via Meta-learning Agentic Memory Designs.pdf#page=3&selection=97,10,100,16|Learning to Continually Learn via Meta-learning Agentic Memory Designs, p.3]]
> > Representing memory designs in code also enables interpretability and allows FMs to leverage their prior knowledge acquired during pretraining about agentic systems and coding 

> [!PDF|] [[Learning to Continually Learn via Meta-learning Agentic Memory Designs.pdf#page=3&selection=5,0,9,1|Learning to Continually Learn via Meta-learning Agentic Memory Designs, p.3]]
> > AI-generating algorithms (Clune, 2020) and automated machine learning (Hutter et al., 2019) aim to replace hand-engineered components with automatically learned ones.

> [!PDF|] [[Learning to Continually Learn via Meta-learning Agentic Memory Designs.pdf#page=4&selection=112,25,114,9|Learning to Continually Learn via Meta-learning Agentic Memory Designs, p.4]]
> >  All designs maintain non-zero sampling probabilities, keeping all potential improvements reachable

# 9. Example Design:
```
Inventory:
a: a blessed +1 mace (weapon in hand)
b: a +0 robe (being worn)
...
Stats:
Strength:15/15
Dexterity:10
...

Cursor:Yourself a priestess

Observation:
vertical closed door far westnorthwest
horizontal wall near north and northwest
...

Message:
>Hello Agent, welcome to NetHack!  You are a neutral human Priestess. Your goal is to get as far as possible in the game
```
----------------------------------------------------------------------

**Memory layers:**
- **TaskSchema:** Store and retrieve prior schemas (parsed context) with outcome summaries. 
```
task_schema = {
    "task_id": "minihack:643759935e128b64511fe4c27717958dc1ebf583",
    "goal": "get as far as possible in the game",
    "role": "valkyrie",
   ...
    "local_topology": {"walls_at": ["north","northeast","south","southeast"], ...},
    "map_neighbors": {"north":".", "northeast":".", "east":".", ...},  # all passable
    "similar_cases": [case_1, case_2]
}
```
- **SpatialPriors**: A knowledge graph with nodes being the entities and edge being the relations between them
```
Nodes (entities):  wall, boulder, stairs_up, stairs_down, trap, fountain, monster, ...
Nodes (relations): blocks_movement, pushable, goal_target, hazard, unknown

spatial_priors = [
    "wall -> blocks_movement (support=280.5)"
] 
```

- **StrategyLibrary**: Store strategies and pitfalls, retrieve high-signal guidance for current task --> compress to bullets
```
plan_outline = {
    "plan_title": "Navigating the Grid to Progress",
    "ordered_steps": [
        "Assess initial position and adjacent walls.",
        ...
        "Adapt strategy based on encountered obstacles."
    ],
    "pitfalls_to_avoid": [
        "Rushing movements without assessing surroundings.",
       ...
        "Becoming fixated on one direction despite walls blocking."
    ],
    "key_checks": [
        "Confirm no walls block planned movement direction.",
        "Reassess health and inventory after each turn.",
        "Check if any enemies appear during movement."
    ]
}
```


- **RiskInteration**: Store risk and interaction with inventories, terrain, monsters 
```
risk_tips = [
    "Explore passable directions; avoid blocked paths.",
    ...
    "Use the potion of sickness when HP is low for risky moves."
]
```

- **Reflex**: Store reflex rules from current task schema and spatial priors
```
reflex_tips = [
    "Avoid bumping into blocked directions: north, northeast, south, southeast. Pivot to open tiles.",
    "Walls/bars block movement; navigate around instead of repeating bumps."
]
```

**Memory retrieve flow:**  Task schema -> Spatial priors -> Strategy retrieval -> Risk tips -> Reflex tips
**Memory update flow:** Summarize episode -> Update Spatial KG -> Update Strategy & Risk -> Index Schema