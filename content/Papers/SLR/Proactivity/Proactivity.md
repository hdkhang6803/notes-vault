---
Date: "2025"
Authors: Hermine J. Grosinger
Venue: AAMAS
Paper: The Next Level of Long-Term Agent Autonomy — Proactively Acquiring Knowledge and Abilities
Memory type:
  - Token-level
Agent env: Multi-agent
Record format: Text
Memory architecture:
  - 3-tier
  - graph-a
Tackle Module: Memory design
Need offline initialization: false
Fine-tuning?: false
Other tags:
Tags:
---
## 1. Terminology

| Term                                                       | Definition                                                                                                                                                                                                                                                                     |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Proactivity                                                | A comprehensive, human-like, self-initiated and anticipatory behavior (as opposed to purely reactive behavior), requiring context awareness, prediction of future states, mental simulation of actions, and preference/epistemic reasoning.                                    |
| Proactive Learning                                         | A self-initiated, anticipatory decision process by which an agent determines _when_ to learn _what_ new knowledge or ability, rather than learning a fixed objective given by a human (as in classical ML).                                                                    |
| Epistemic Reasoning                                        | The capability enabling an agent to reason about its own and other agents' (including humans') knowledge and beliefs, underpinned by the Introspection Axioms.                                                                                                                 |
| Introspection Axioms                                       | Two axioms: <br>+ positive introspection ($⊨ K_iφ ⇒ K_iK_iφ$: if agent i knows φ, i knows that it knows φ)<br>+ negative introspection ($⊨ ¬K_iφ ⇒ K_i¬K_iφ$: if i does not know φ, i knows that it does not know φ)<br>that let an agent recognize gaps in its own knowledge. |
| Theory of Mind (ToM)                                       | The general capacity, studied in both psychology and AI, to reason about other agents' mental states (knowledge, beliefs).                                                                                                                                                     |
| Epistemic Logic (EL)                                       | A family of logics for reasoning about knowledge/belief, classically built on Kripke possible-worlds semantics, where an agent believes what holds in all worlds it considers accessible.                                                                                      |
| Dynamic Epistemic Logic (DEL)                              | The dynamic extension of EL that updates an epistemic model M with an event model E via a product update (M ⊗ E = M′), capturing both ontic (world-state) and epistemic (belief-state) change from an action.                                                                  |
| Belief-set operations (expansion / contraction / revision) | Three operations for updating a belief set encoded as sentences in a language L: expansion accepts new information φ; contraction removes existing information φ; revision accepts φ while removing anything inconsistent with it.                                             |
| Ability                                                    | Defined as an abstract skill — a high-level behavior composed of possibly-learnable low-level actions — understood as "knowing how" to achieve a goal via a sequence of actions.                                                                                               |
| Propositional Dynamic Logic (PDL)                          | An early formalism where [a]p denotes "when program (action) a terminates, assertion p holds," used to reason about action effects.                                                                                                                                            |
| Alternating-time Temporal Epistemic Logic (ATEL)           | A logic combining ATL (alternating-time temporal logic) with epistemic notions, using the ⟨⟨a⟩⟩ modality ("bringing about") so that K_a⟨⟨a⟩⟩◇φ expresses that agent a knows it can ensure φ eventually holds.                                                                  |
| Constructive Strategic Logic (CSL)                         | A logic extending AT(E)L that distinguishes "knowing that" (operator K) from "knowing how" (operator 𝕂, i.e., knowing which strategy achieves a goal).                                                                                                                        |
| Knowing-how operator Kh(ψ, φ)                              | A formalism (Wang; Areces et al.) expressing that an agent knows how to achieve goal φ given precondition ψ if a plan exists that, executed in any ψ-state, ends in a φ-state.                                                                                                 |
| Epistemic Planning (EP)                                    | Planning over both ontic and epistemic states/actions, used to construct policies whose constituent actions the agent may still need to learn.                                                                                                                                 |
| Dynamic Logic for Learning Theory (DLLT)                   | A logic (Baltag et al.) with modality [o]φ ("after evidence o is observed, φ holds") and a learning operator L(ō) mapping an observation sequence ō to the agent's strongest resulting conjecture.                                                                             |
| Value Alignment (VA)                                       | The requirement that an autonomous agent's goals and behavior align with human values, raised here as an ethical risk of high-autonomy proactive-learning agents (e.g., "wire-heading," where an agent optimizes a proxy reward at the expense of what humans actually want).  |

---

## 2. Framing / Scope

The paper positions itself as extending prior work on **proactive acting** (self-initiated, anticipatory behavior) toward a new, largely unaddressed problem: **proactive learning** (an agent's self-initiated, anticipatory decision about _when_ to acquire _what_ new knowledge or ability).

- **Grounding definition of proactivity:** adopts an extrinsic view (following Grosinger and Lorini) — autonomously initiating action while accounting for future state development and anticipating effects on other agents' minds and the environment — rather than an intrinsic-drives (BDI) view.
- **Motivation ("why proactive learning"):** long-term agents operating in dynamic, real-world environments must be able to learn new knowledge and/or new abilities autonomously as the world evolves.
- **Methodological stance ("why logic"):** the paper deliberately focuses on **formal/logic-based methods** for the reasoning ("what to learn, when, why") while suggesting these can be paired with machine learning (e.g., reinforcement learning) for the "how"
- **Scope boundary:** the paper explicitly narrows from general cognitive abilities (context awareness, anticipation, mental simulation, preference reasoning) to the three it considers most crucial for proactive learning specifically: 
	- epistemic reasoning (reasoning if model knows the knowledge)
	- reasoning on abilities (reasoning whether model knows how to achieve the goal)
	- learning 

---

## 3. Categories

The paper organizes its contribution into three challenges, each surveyed with candidate formal tools:

**I. Proactive Knowledge Learning**: theory/methods for an agent to decide _when_ to acquire _what_ new **knowledge**.
- Built on epistemic reasoning (Introspection Axioms) so the agent can recognize what it does/doesn't know.
- Candidate formalisms: 
	- Epistemic Logic (Kripke models)
	- belief sets (expansion/contraction/revision)
	- Bayesian epistemic states
	- Dynamic Epistemic Logic (product update)
	- causal-epistemic combinations (causal Kripke models, structural causal models)
	- Dynamic Logic for Learning Theory (DLLT).
- Illustrated with a running example (ProRo/Bob): combining causal reasoning ("not knowing medicine → not taking it → illness") with epistemic reasoning to decide to learn the medicine's location.

**II. Proactive Ability Learning**: theory/methods for an agent to decide _when_ to acquire _what_ new **ability** (abstract skill/strategy).
- Candidate formalisms: 
	- Propositional Dynamic Logic (PDL)
	- Alternating-time Temporal Epistemic Logic (ATEL)
	- Constructive Strategic Logic (CSL)
	- knowing-how logics (Wang; Areces et al.)
	- game-theoretic causal reasoning (Hammond et al.)
	- probabilistic temporal logic (Motamed et al.).
- Proposed neuro-symbolic pipeline: formal reasoner infers (i) _if_ to learn a new ability, (ii) _what_ ability, (iii) _when_; then ML (e.g., reinforcement learning) learns (iv) _how_ to execute it.
- Epistemic Planning (EP) proposed as a bridge for finding policies whose constituent actions may need to be learned.

**III. Unified Theory of Proactive Learning**: integrating I and II, since knowledge and ability learning can be mutually dependent.

- **Knowledge → Ability:** to learn ability σ, the agent may first need to learn knowledge φ necessary for it 
>	example: ProRo must learn the door code before it can learn the ability of picking up and bringing Ann's lunch.
- **Ability → Knowledge:** to learn knowledge φ, the agent may first need to learn an ability σ that lets it retrieve φ 
>	example: ProRo must learn the ability to open drawers to discover which drawer holds Bob's medicine.
- The paper notes these dependencies can chain arbitrarily (learn knowledge → learn ability → learn further knowledge).

---

## 4. Findings

- The paper's core claim is that **proactive acting** (self-initiated, anticipatory action) is being addressed in current research, but **proactive learning** (self-initiated, anticipatory decisions about what/when to _learn_) is a "completely unexplored field."
- Epistemic reasoning is presented as the necessary enabling capability for an agent to recognize (a) that it lacks knowledge, and (b) which knowledge it lacks.
- Learning new abilities is framed analogously: the agent needs epistemic reasoning to recognize which ability it lacks (i.e., which goal it does not know how to reach), and a knowing-how-style logic to represent abilities as strategies/plans.
- Knowledge learning and ability learning are argued to be **intertwined and potentially bidirectional dependencies**, motivating a unified theory.
- The proposed approach throughout is **neuro-symbolic**: formal/logic-based reasoning handles the high-level decisions (if/what/when to learn), while machine learning (e.g., reinforcement learning) is delegated the low-level "how" (e.g., motor skill acquisition).
- The paper raises the combination of **causal and epistemic reasoning** as an underinvestigated but promising basis for deciding what knowledge to learn.

---

## 5. Discussed Gaps

- **Causal-epistemic integration is underinvestigated:** existing causal reasoning approaches (standard causal models, causal Kripke models, structural causal models, action influence models) have not been substantially combined with epistemic reasoning for proactive learning purposes.
- **No unified theory yet exists** integrating proactive knowledge learning and proactive ability learning (Challenge III is posed as future work, not solved).
- **Scalability/cost:** the surveyed formal-logic methods are described as expressive but computationally costly; only limited proposals (e.g., RP-MEP, a KD45-based epistemic planner that limits belief-nesting depth) address efficiency, and the paper flags this as an open area.
- **No benchmarks exist** for comparing proactively acting/learning agents e.g., no way to evaluate whether an agent choosing to learn ability σ₁ and knowledge φ₁ behaves better or worse than one choosing σ₂ and φ₂ in the same situation.
- **Value alignment risk:** high autonomy in deciding what to learn raises the risk of "wire-heading" (optimizing a proxy objective while violating actual human intent/dignity), which the paper illustrates but does not solve.
- **Transparency in neuro-symbolic integration:** while knowledge-based methods are noted as inherently more transparent (human-readable, white-box) than the ML components used for "how," the paper does not specify concrete mechanisms for ensuring transparency where the two are combined.

---

## 6. Future Directions

- Develop a **theory and computational methods for Challenge I** (proactive knowledge learning) and **Challenge II** (proactive ability learning) independently, then **integrate them (Challenge III)** into one unified theory and computational framework.
- Further investigate the **combination of causal and epistemic reasoning** to determine what new knowledge an agent should learn.
- Explore **qualitative planning** (in addition to epistemic planning) as an alternative route to reasoning about action effects for ability learning.
- Build on existing **scalability-oriented proposals** (e.g., RP-MEP's belief-nesting/formula restrictions) to make the proposed formal methods computationally tractable at scale.
- Develop **benchmarks** for comparing the behavior of different proactively acting and proactively learning agents.
- Investigate **Value Alignment** mechanisms suited to proactive learning agents, e.g., Transparent VA (human feedback to verify/amend the value-learning process) and modeling agent uncertainty over human preferences (per Russell) so agents remain "open" to correction.
- Address **trustworthiness/transparency** requirements (per EU HLEG AI ethics guidelines) specifically for neuro-symbolic proactive-learning architectures, ensuring humans retain the ability to override agent-initiated learning decisions.

---

## 7. Memory/Context Related

N/A — this paper is about an agent's **epistemic state** (what it knows/believes, modeled via Kripke models, belief sets, or Bayesian distributions) and **ability state** (what it knows how to do), reasoned over with formal logic in a multi-agent/robotics setting. It does not address memory or context architectures for LLM-based systems (no discussion of context windows, retrieval, compression, or neural memory storage). The closest tangential link is its treatment of _belief update operations_ (expansion/contraction/revision) and DEL's product update, which are conceptually related to "state update" mechanisms but are not applied to LLM context/memory in this paper.

---

## 8. Highlights

> [!PDF|255, 208, 0] [[Proactivity.pdf#page=1&annotation=557R|Proactivity, p.1]]
> > human proactivity established in organizational psychology, as being anticipatory and self-initiated action to impact people and their environment

> [!PDF|255, 208, 0] [[Proactivity.pdf#page=1&annotation=560R|Proactivity, p.1]]
> > It has been found that proactive AI systems are preferred [ 5], more easily accepted [ 38 ], and trusted [ 31 ] by humans

> [!PDF|255, 208, 0] [[Proactivity.pdf#page=2&annotation=563R|Proactivity, p.2]]
> > While unsolved, long-term proactive acting is being addressed today, whereas poactive learning is a completely unexplored field

> [!PDF|255, 208, 0] [[Proactivity.pdf#page=2&annotation=572R|Proactivity, p.2]]
> >  Grosinger [22] and Lorini [33] who speak of behavior that is autonomously initiating action taking into account future state development as well as anticipating potential consequences of the agent’s actions on (the mind) of other agents and the environmen

> [!PDF|255, 208, 0] [[Proactivity.pdf#page=3&annotation=608R|Proactivity, p.3]]
> > fusion of causal models and Kripke models, thus using possible world semantics, which they call Causal Kripke Models. 


> [!PDF|255, 208, 0] [[Proactivity.pdf#page=3&annotation=617R|Proactivity, p.3]]
> > ability we mean abstract skill.

> [!PDF|255, 208, 0] [[Proactivity.pdf#page=3&annotation=638R|Proactivity, p.3]]
> >  high-level reasoner proactively infers (i.) if to learn a new ability, (ii.) what new ability should be learned and (iii.) when

