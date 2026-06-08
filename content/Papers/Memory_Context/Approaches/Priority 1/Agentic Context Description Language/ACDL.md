---
Tags:
Date: "2026"
Authors: Noga Peleg Pelc, Gal A. Kaminka, Yoav Goldberg
Venue: CAIS
Paper: A Language for Describing Agentic LLM Contexts
Memory type:
  - Token-level
Agent env:
Record format: Text
Memory architecture:
  - Language abstraction
Tackle Module: Context abstraction
Need offline initialization: false
Fine-tuning?: false
Other tags:
---
# 1. Terminology

| Term                | Definition                                                                                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ACDL                | Agentic Context Description Language: a DSL for precisely specifying the structure and temporal evolution of LLM input contexts                                                            |
| Role Message        | A single message labeled with one of four roles: System (S), User (U), Assistant (A), or Tool (T)                                                                                          |
| Context Engineering | The macro-level design of _what_ information is presented to the LLM, _how_ it is packed into role messages, and _where_ it is placed, distinct from prompt engineering (wording/phrasing) |
| Time Step (`@T`)    | A discrete clock tick in the agent's execution; indexed with `@` prefix (e.g., `@T` = current, `@T-1` = previous)                                                                          |
| Sub-step (`@T.I`)   | A nested time index within a main step, used for multi-scale loops (e.g., a ReAct inner loop within a chat outer loop)                                                                     |
| Information Source  | The provenance category of a context element: constant template, `sys` (internal state), `env` (environment/external), `resp` (prior LLM output), or function                              |
| Template            | An `ALL_CAPS` placeholder representing a fixed string whose exact wording is out of scope for ACDL                                                                                         |
| Fragment            | A reusable, parameterized sub-specification: either a `StrFrag` (content-level) or `RolesFrag` (message-level)                                                                             |
| `ForEach`           | ACDL control construct for iterating over time ranges or collections to unpack history into the context                                                                                    |
| `PromptEndsHere`    | An early-termination marker signaling that no further messages are appended when a given condition holds                                                                                   |
| Mark Block          | A purely visual annotation (`Mark N { ... }`) that places a numbered bracket in the rendered diagram for cross-referencing in prose                                                        |
| Context Compaction  | A design pattern where older history is summarized rather than kept verbatim to manage context length                                                                                      |

---
# 2. Paper Summary (What)

The paper introduces **ACDL (Agentic Context Description Language)**, a formal domain specific language (DSL)for describing how LLM **input contexts** are constructed and evolve across interaction steps in agentic systems. ACDL abstracts away exact prompt wording and replaces it with symbolic labels encoding the _role_, _type_, and _source_ of each context element, combined with time-step indexing and control flow. The language is purely descriptive, it specifies what the context contains at each step, not how the underlying agent or tools are implemented.

---
# 3. What it Solves (Why)

-  No standard exists to describe how LLM context is structured and evolves across interaction steps 
- Informal prose and ad-hoc diagrams fail to capture prompt dynamics or enable systematic comparison 
- Reproducing or re-implementing published agents is difficult because context assembly logic is rarely explicit in papers 
- Within teams, communicating incremental context changes rigorously is impractical without a shared language

---

# 4. Methodology (How)


ACDL is built around four composable primitives, illustrated here with a single running example
>
>a multi-turn ReAct agent that progressively grows in expressiveness.

**Role messages and information sources.** The most basic unit is a role-annotated message containing typed information pieces. Sources are distinguished syntactically: 
- `ALL_CAPS` denotes constant templates
- `env.*` denotes environment state
- `sys.*` denotes internal system state
- `resp.*` denotes prior LLM outputs.

```
ReactBase[@T]: {
    S: INSTRUCTIONS
    S: AVAILABLE_TOOLS
    U: env.user_input[@1]
}

// This describes a static, one-shot context: a system instruction block, a tool list, and the initial user query. No history is present.
```

**Time indexing and `ForEach` for history unpacking.** Real agentic systems accumulate history across steps. ACDL introduces `@T` for the current step and `ForEach` for iterating over past steps, making the exact structure of accumulated history explicit.

```
ReactBase[@T]: {
    S: INSTRUCTIONS
    S: *AVAILABLE_TOOLS
    U: env.user_input[@1]
    // history
    ForEach(@t: range(2, @T-1)) {
        A: { 
	        resp.tool_reasoning[@t]
	        sys.tool_used[@t] 
        }
        T: sys.tool_used[@t].tool_response
    }
    S: *USE_TOOLS_TO_SOLVE_TASK
}
```

Now the context at step `@T` includes all previous reasoning-action-observation triples. The `ReactNoReasoningInHistory` variant simply drops `resp.tool_reasoning[@t]` from the inner `A:` block — a one-line change that ACDL makes immediately visible and discussable.

**Sub-steps and `PromptEndsHere` for multi-scale loops.** When an agent has both an outer chat loop (user turns) and an inner ReAct loop (tool-call steps), two nested time indices are needed. `@T.I` denotes the `I`-th substep of the `T`-th main step. `PromptEndsHere` marks where the prompt truncates on the initial substep.

```
React2[@T.I]: {
    S: INSTRUCTIONS_AND_TOOLS
    // Chat loop (Main turns)
    ForEach(@t : range(1, @T)) {
        U: env.user_question[@t]
        PromptEndsHere when (@T == @t && @T.0)
        
        // React loop (inner substeps)
        ForEach (@i : range(1, @t.substeps)) {
            A: sys.tool_used[@t.i].name_and_args
            T: sys.tool_used[@t.i].tool_response
        }
        
        PromptEndsHere when (@T == @t && @T.I)
        A: resp.response  // final answer after ReAct loop
    }
    
    // Current turn React loop
    ForEach (i : 1 ... @T.substeps {
        A: sys.tool_used[@T.i].name_and_args
        T: sys.tool_used[@T.i].tool_response
    }
}
```

This single specification captures both temporal scales and their interaction without ambiguity.

**Fragments and Mark blocks for modularity and communication.** 
- Repeated context patterns are encapsulated in `StrFrag` (generate content without roles) or `RolesFrag` (produce complete role block) fragments, invoked with `Frag` keyword. 
- Mark blocks attach numbered brackets to regions of the diagram (representational only).

```
RolesFrag ToolTurn[@t]: {
    A: { resp.tool_reasoning[@t]
         sys.tool_used[@t] }         ]1
    T: sys.tool_used[@t].tool_response
}

React2Modular[@T.I]: {
    S: INSTRUCTIONS_AND_TOOLS
    ForEach(@t : range(1, @T - 1) {
        U: env.user_question[@t]
        ForEach (@i : range(1, @t.substeps) {
            Frag ToolTurn[@t.i]
        }
        A: resp.response
    }
    PromptEndsHere when (@T == @t && @T.0)
    U: env.user_question[@T]
    ForEach i : 1 ... @T.substeps {
        Frag ToolTurn[@T.i]
    }
}
```

Syntax summary

| Category           | Element             | Syntax / Convention                         | Notes                                                                                |
| ------------------ | ------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Spec Structure** | Prompt definition   | `PromptName[idx1, ...]: { <blocks> }`       | Multiple prompts allowed per file                                                    |
|                    | String fragment def | `StrFrag Name[params]: { <content> }`       | Produces content pieces, no role                                                     |
|                    | Role fragment def   | `RolesFrag Name[params]: { <blocks> }`      | Produces complete role messages                                                      |
| **Roles**          | System              | `S:`                                        | Instructions, persona, constraints                                                   |
|                    | User                | `U:`                                        | External inputs, observations                                                        |
|                    | Assistant           | `A:`                                        | Prior outputs, reasoning, actions                                                    |
|                    | Tool                | `T:`                                        | Structured tool call results                                                         |
|                    | None (legacy)       | `N:`                                        | Completion format only; cannot coexist with other roles                              |
| **Role Form**      | Single-line         | `U: env.x[@T]`                              | Exactly one element; no control flow permitted                                       |
|                    | Multi-line          | `U: { ... }`                                | Any combination of elements and control flow                                         |
|                    | Nesting             | —                                           | Role messages cannot be nested inside other role messages                            |
| **Namespaces**     | Environment         | `env.path[idx]`                             | External inputs, observations, world state                                           |
|                    | System              | `sys.path[idx]`                             | Internal agent state, memory, tool history                                           |
|                    | Response            | `resp.path[idx]`                            | Prior LLM outputs, reasoning traces                                                  |
|                    | No-index variable   | `sys.agent_desc`                            | Time-invariant constant                                                              |
| **Indices**        | Current main step   | `@T`                                        | Uppercase = current                                                                  |
|                    | Relative main step  | `@T-1`, `@T+1`                              | Arithmetic permitted: `+`, `-`, `*`, `/`, `%`                                        |
|                    | Current sub-step    | `@T.I`                                      | Sub-step `I` of current main step `T`                                                |
|                    | Past sub-step       | `@t.i`                                      | Sub-step `i` of past turn `t` (in loops)                                             |
|                    | Sub-step count      | `@t.substeps`                               | Total sub-steps in turn `t`                                                          |
|                    | Variable depth      | `@T.*`                                      | Arbitrary nesting shorthand                                                          |
|                    | Non-time index      | no `@` prefix                               | Named entity, collection key, or dimension                                           |
| **Naming**         | Template            | `ALL_CAPS`                                  | Fixed string; content defined outside ACDL                                           |
|                    | Function            | `camelCase(args)`                           | Computed content; assumed pure                                                       |
|                    | Variable path       | `dot.separated.names`                       | —                                                                                    |
|                    | Identifiers         | Start with letter or `_`                    | Case-sensitive; letters, digits, `_` only                                            |
| **Templates**      | With arguments      | `INSTRUCTIONS(sys.conf.role, sys.time[@1])` | Arguments embedded into the string                                                   |
| **Functions**      | General             | `fnName(arg1, arg2, ...)[indices]`          | Args can be variables, literals, expressions, nested calls                           |
|                    | Built-in range      | `range(start, stop, step?)`                 | Exclusive upper bound; `step` defaults to 1                                          |
| **Control Flow**   | Loop                | `ForEach(var: iterable) { <body> }`         | Iterable is `range(...)` or a collection variable                                    |
|                    | Conditional         | `If <cond> { } ElseIf <cond> { } Else { }`  | Operators: `==` `!=` `<` `>`; connectives: `&` `\|`                                  |
|                    | Switch              | `Switch expr { Case val { } Default { } }`  | Selects among multiple alternatives                                                  |
|                    | Loop control        | `break`, `continue`                         | Standard semantics inside loops                                                      |
|                    | Placement           | —                                           | May appear at top level (gates role messages) or inside a role block (gates content) |
| **Early Exit**     | PromptEndsHere      | `PromptEndsHere when <condition>`           | No further messages appended when condition holds                                    |
| **Fragments**      | String invocation   | `Frag Name[args]` inside a role block       | Expands content in place, inherits enclosing role                                    |
|                    | Role invocation     | `Frag Name[args]` at top level              | Expands to full sequence of role messages                                            |
|                    | Kind inference      | —                                           | Parser determines kind from invocation context                                       |
| **Marks**          | Mark block          | `Mark N { <blocks> }`                       | Visual bracket `]N` in rendering; no semantic effect                                 |
| **Names**          | Binding             | `Name var := expression`                    | Any valid ACDL expression                                                            |
|                    | Reference           | `$var`, `$var.field`, `$var[i]`             | `$` prefix required                                                                  |
|                    | List comprehension  | `Name x := [expr for v in range(...)]`      | Constructs a list; same `$x` reference syntax                                        |
| **Comments**       | Syntax              | `// text`                                   | Remainder of line ignored                                                            |
|                    | Inline              | After a content element on same line        | Renders beside the element in the diagram                                            |
|                    | Standalone          | On its own line                             | Renders at current nesting level                                                     |

---

## 5. Benchmarks

### 5.1. Other Baselines

There are no directly competing systems evaluated. The paper positions ACDL against three related languages but these are compared conceptually rather than empirically: 

| [[PromptML]], [[PDL (IBM)]], [[POML (Microsoft)]]                                           | ACDL                                                                                                                           |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| target single-prompt organization or executable pipeline orchestration                      | <                                                                                                                              |
| focus on organizing the content of individual prompts or on executing LLM-powered workflows | focus on describing the mapping of temporally evolving system state and history to LLM contexts across multi-turn interactions |

### 5.2. Benchmarks
N/A
### 5.3. Notable Results
N/A

---

## 6. Strengths

- **Implementation-agnostic abstraction layer**: it describes _what_ the context contains without prescribing how it is assembled, making it portable across frameworks and runtimes.
- **Scales to real-world complexity:** The paper demonstrates ACDL on OpenCode, OpenClaw, and the Gemini Pokémon Blue agent showing that the language is not limited to toy ReAct examples.
- **Dual-mode utility:** ACDL serves both human-to-human communication (whiteboards, papers) and human-to-AI communication (Claude Code was shown to implement agentic loops from ACDL specs), with the latter being a promising frontier.

---

## 7. Gaps

- **Mutable-state systems remain awkward to encode:** ACDL assumes state is immutable within a construction step, requiring an equivalent reformulation before description
- **Asynchronous multi-agent systems lack clean support:** Agents with independent, unsynchronized clocks sharing mutable state are unrepresentable without an inelegant explicit synchronization workaround, yet this architecture is increasingly relevant for production multi-agent deployments.

---

# 8. Highlights
> [!PDF|255, 208, 0] [[ACDL.pdf#page=4&annotation=539R|ACDL, p.4]]
> > At the most abstract description, an LLM context is a linear sequence of information pieces

> [!PDF|255, 208, 0] [[ACDL.pdf#page=4&annotation=542R|ACDL, p.4]]
> > ACDL is concerned with describing the queries (contexts) sent to the LLM by an agentic controller.

> [!PDF|255, 208, 0] [[ACDL.pdf#page=4&annotation=545R|ACDL, p.4]]
> > sys captures state maintained by the system (configuration, memory, action and tool-use history, etc.), while env captures external state observed from the environment (user inputs, observed world state, etc.

> [!PDF|255, 208, 0] [[ACDL.pdf#page=5&annotation=548R|ACDL, p.5]]
> >  Functions have descriptive names, and their semantics are either inferred by the reader or specified elsewhere (not in ACDL).

> [!PDF|255, 208, 0] [[ACDL.pdf#page=5&annotation=566R|ACDL, p.5]]
> > Functions in ACDL are assumed to be pure, and their return values depend only on their parameters and external constants:

> [!PDF|255, 208, 0] [[ACDL.pdf#page=5&annotation=579R|ACDL, p.5]]
> > Note that the very last step of the process, in which the LLM responds with the final answer which is returned to the user, is not part of the ACDL context:

> [!PDF|255, 208, 0] [[ACDL.pdf#page=5&annotation=587R|ACDL, p.5]]
> > The system’s clock may progress due to: (a) external events from the environment (like the arrival of user input; (b) internal timers for periodic checking of state ("hearbeats"); or (c) the conclusion of some internal process that necessitates a state change (the conclusion of some actions or tool calls).

> [!PDF|255, 208, 0] [[ACDL.pdf#page=5&annotation=590R|ACDL, p.5]]
> > The signature React2[@T.I] indicates the context at the 𝐼 th substep of the 𝑇 th main step.

> [!PDF|255, 208, 0] [[ACDL.pdf#page=5&annotation=593R|ACDL, p.5]]
> > We can make this explicit with PromptEndsHere markers


> [!PDF|255, 208, 0] [[ACDL.pdf#page=6&annotation=596R|ACDL, p.6]]
> > ACDL Context definition can be parameterized not only by the time step, but also by a variable indicating its agent id. 

> [!PDF|255, 208, 0] [[ACDL.pdf#page=7&annotation=604R|ACDL, p.7]]
> > ACDL makes context nuances easy to locate and communicate.

> [!PDF|255, 208, 0] [[ACDL.pdf#page=7&annotation=607R|ACDL, p.7]]
> > ACDL helps document and clarifies complex contexts

