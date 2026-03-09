# Aptitude Inclination Indicator (Aii)
### Methodology Document — Version 2.1

---

## 1. What the Aii Is

The Aptitude Inclination Indicator is a digital, scenario-based assessment tool designed to surface a person's instinctive cognitive predispositions — the mental tools they reach for first when confronted with a problem, before effort, training, or strategy kicks in.

It does not measure what someone *can* do. It does not test performance, speed, or correctness. Its output is a **Perceived Predisposition Profile**: a picture of which cognitive approaches a person instinctively gravitates toward, across a variety of realistic situations.

This distinction matters. The Aii is honest about what it is: a structured self-perception tool, not a psychometric capacity test. Its value lies in initiating self-reflection, informing career and learning conversations, and pointing toward skill domains where a person's natural inclinations are likely to be assets.

---

## 2. What the Aii Is Not

The Aii is not a substitute for validated psychometric instruments such as the O\*NET Ability Profiler or the General Aptitude Test Battery (GATB). Those tools measure demonstrated capacity under controlled conditions. The Aii measures expressed affinity and instinctive approach under no performance pressure.

Scores should be treated as **directional indicators**, not ceilings. A high score on an aptitude dimension means the person consistently and instinctively reached for that cognitive tool. A low score means they did not — not that they are incapable of using it.

---

## 3. The Aptitude Framework

The Aii assesses six aptitude dimensions. These are defined as natural affinities and raw potential — largely stable capacities that dictate the ceiling and speed of learning in a given field. They are distinct from traits (how you behave), values (what you want), and skills (what you have already learned).

### G — Logical and Analytical Reasoning
Capacity to follow and construct logical arguments, identify cause and effect, and reason systematically from premises to conclusions. Determines how quickly someone can learn formal systems, rules, and structured domains such as law, logic, and systems design.

*In an AI-augmented environment:* Routine logical execution is increasingly automated. High-level reasoning, system design, and oversight of AI output still require this capacity.

### N — Quantitative and Numerical Facility
Capacity to work comfortably with numbers, quantities, proportions, and data. Determines ceiling and learning speed in math-heavy fields, data analysis, finance, and technical domains that rely on numerical reasoning.

*In an AI-augmented environment:* Routine computation is highly automated. Interpretation, judgment on metrics, and strategic use of quantitative insight remain human tasks.

### V — Verbal and Linguistic Facility
Capacity to understand and use language with precision and nuance: reading, writing, and grasping subtle meaning. Determines ceiling in communication-heavy roles, persuasion, editing, and domains where wording and interpretation matter.

*In an AI-augmented environment:* Nuance, persuasion, and precise communication remain distinctly human. AI agents take instructions literally — human verbal facility is needed to frame, edit, and judge their output.

### P — Pattern Recognition and Abstraction
Capacity to spot similarities, analogies, and recurring structures across different contexts. Supports learning by transfer and cross-domain thinking; determines how quickly someone can see "this is like that" and apply prior knowledge in new settings.

*In an AI-augmented environment:* Cross-domain synthesis and seeing connections across fields favor humans. AI is strong within a domain but weaker at structural transfer.

### T — Technical / Hands-on Orientation
Natural inclination to understand how tools, machines, or software systems function internally by reasoning about their components and cause-and-effect relationships. Best positioned toward system design, architecture thinking, and supervision of technical systems rather than routine execution.

*In an AI-augmented environment:* Routine implementation is increasingly automated. The ability to understand system behavior, diagnose problems, and design or supervise technical systems remains valuable.

### C — Creative and Open-ended Thinking
Capacity for imagination, divergent thinking, and generating options when there is no single right answer. Valued where human taste, novelty, and judgment matter. Supports learning in arts, design, and strategy.

*In an AI-augmented environment:* Judgment, novelty, and open-ended creation are harder to automate. Humans directing for taste remain in the loop; higher-value tasks increasingly favor this capacity.

---

## 4. The Triangle Interface

The Aii uses an equilateral triangle as its core interaction mechanism.

Each vertex of the triangle represents one aptitude dimension, identified by a short label (e.g., "The Auditor") and a first-person behavioral descriptor that concretely describes what a person with that instinct would do in the given scenario.

A movable token is placed at the centre of the triangle. The user reads a scenario prompt and drags the token toward the vertex — or toward a point between vertices — that best represents their first instinct.

This mechanism allows for three types of response:

- **Discrete choice:** Dragging directly to a vertex indicates a strong, singular instinctive predisposition toward one aptitude.
- **Hybrid choice:** Dragging to a point along an edge indicates a perceived combination of two aptitudes, weighted by proximity.
- **Blended choice:** Dragging to a point within the triangle interior indicates a mix of all three aptitudes in varying proportions.

The final position of the token is recorded as an (x, y) coordinate, which is converted to barycentric coordinates representing the relative weight of each aptitude in that response.

---

## 5. Scoring

Each response generates a barycentric coordinate for each aptitude vertex in that triangle. The coordinate ranges from 0 (token at the opposite edge) to 1 (token placed exactly at that vertex).

The score for a single aptitude in a single triangle is:

```
score = (barycentric_coordinate × 4) + 1
```

This maps the [0, 1] range to a [1, 5] scale.

Across the full battery of 8 scenarios, each aptitude appears in multiple triangles. The final score for each aptitude is the **simple average** of all individual triangle scores for that dimension.

The result is a six-dimension Perceived Predisposition Profile with each aptitude scored on a 1–5 scale.

---

## 6. Scenario Design Principles

The scenarios are the most critical component of the Aii. They must elicit behavioral proxy responses — choices that reveal the aptitude instinct — rather than simply asking people to report their preferences.

Four principles govern every scenario:

**1. All three options must be equally valid.** No vertex should represent the "right" answer, the "smart" answer, or the professionally expected answer for the context. If one option is obviously superior, the scenario measures judgment or social desirability, not instinct.

**2. The prompt must not prime any specific aptitude.** The framing of the scenario should be neutral across all three vertices. If the prompt uses words like "creative," "analytical," or "systematic," it will anchor responses before the user reads the options.

**3. Each option must describe a concrete action, not a stated preference.** Descriptors are written in first person and describe what the person would *do*, not what they *value* or *enjoy*. "I'd map the process step by step" is a behavioral proxy. "I prefer structured approaches" is a stated preference. Only the former reveals instinct.

**4. The options must feel personally recognizable.** If a respondent cannot imagine themselves in the scenario, or cannot connect any of the three options to their own experience, the response will be arbitrary. Scenarios are grounded in universally relatable situations — a tool to learn, a decision to make, a result that surprised you — rather than role-specific professional contexts.

---

## 7. The Scenario Battery

The battery consists of 8 scenarios, each presenting a different situation with three aptitude vertices. The recommended delivery sequence is designed to vary the cognitive register between consecutive scenarios and separate the two diagnostically similar prompts (Scenario 1 and Scenario 7).

**Recommended sequence:** 1 → 3 → 6 → 2 → 4 → 7 → 5 → 8

---

### Scenario 1 — The Broken Process
*G · T · P*

> Your team has been using the same workflow for months. Recently things keep going wrong — deadlines missed, outputs inconsistent — but nobody can agree on why. You have been asked to figure it out. What is your first move?

| Vertex | Label | Instinct |
|---|---|---|
| G | The Auditor | Map the process step by step and find where the logic breaks down |
| T | The System Tracer | Look at the tools and systems — something in how they're configured is causing it |
| P | The Trend Spotter | Look across the failures for what they have in common |

*Why this works:* Three genuinely distinct diagnostic approaches. The signal is whether someone decomposes logic, investigates the system, or hunts for structural similarity.

---

### Scenario 2 — The Vague Brief
*V · C · G*

> A client sends you a one-paragraph brief for a project. It is short, enthusiastic, and completely unclear about what they actually want. You have one reply before a meeting tomorrow. What do you do?

| Vertex | Label | Instinct |
|---|---|---|
| V | The Clarifier | Read it carefully, identify which words carry the ambiguity, ask targeted questions |
| C | The Interpreter | Make a best imaginative guess, sketch two or three directions, present them as options |
| G | The Scoper | Identify what decisions must be made before work can start, structure the reply around those |

*Why this works:* The strongest V vs C separator in the battery. The verbal instinct locates the problem in the language itself. The creative instinct bypasses the ambiguity by generating options. The logical instinct maps the decision dependencies.

---

### Scenario 3 — The New Tool
*T · V · C*

> Your company has just adopted a new project management platform. Everyone has been given access. How do you approach learning it?

| Vertex | Label | Instinct |
|---|---|---|
| T | The System Mapper | Go straight into settings, permissions, and integrations to understand how the parts connect |
| V | The Documentation Reader | Find the official guide and read it carefully before touching anything |
| C | The Doer | Pick a real piece of work and just start — learn by doing, not by studying |

*Why this works:* Low-stakes, universally relatable scenario. Technical means understanding system internals before use. Verbal means extracting meaning from documentation. Creative means preferring output over comprehension.

---

### Scenario 4 — The Argument
*G · V · N*

> Two colleagues are in disagreement about which approach to take on a project. Both make reasonable-sounding cases. You are asked to weigh in. What do you naturally do?

| Vertex | Label | Instinct |
|---|---|---|
| G | The Referee | Ask each to state their assumptions — the disagreement is probably a hidden difference in premises |
| V | The Translator | Listen for whether they're using key terms differently — they may be describing the same thing |
| N | The Measurer | Ask what can actually be measured or compared — let data settle or inform the question |

*Why this works:* The strongest G vs V separator in the battery. Both involve language and argument but through entirely different lenses — premises vs terminology. The N option provides a clean third path that rejects the argumentative frame altogether.

---

### Scenario 5 — The Unexpected Result
*C · P · T*

> You followed a process carefully — a recipe, an experiment, a plan — and got a result you did not expect. Not a disaster, just not what you predicted. What do you do first?

| Vertex | Label | Instinct |
|---|---|---|
| C | The Tinkerer | Change one thing and try again — run a variation and see if the result shifts |
| P | The Pattern Matcher | Think about whether this kind of result has appeared before, in a different context or domain |
| T | The Retracer | Go back through exactly what was done, step by step, and find where the process diverged |

*Why this works:* Universally relatable — everyone has experienced an unexpected result. None of the three options is primed by the prompt. The only triangle where C, P, and T appear together.

---

### Scenario 6 — The Uncertain Choice
*N · G · C*

> You need to make an important decision — it could be a career move, a big purchase, or a personal commitment. You have some information but not enough, and the signals are mixed. How do you think it through?

| Vertex | Label | Instinct |
|---|---|---|
| N | The Modeller | Lay out the key variables, assign rough probabilities, quantify the options to compare them |
| G | The Condition Mapper | Work out what would have to be true for each option to be the right call |
| C | The Story Builder | Picture each option as a concrete future — see which story feels more real and right |

*Why this works:* Personal framing removes seniority and professional identity bias. The only triangle in the battery pairing N directly against C. Three genuinely distinct approaches to uncertainty that map cleanly to their respective aptitudes.

---

### Scenario 7 — The Miscommunication
*V · P · T*

> A project has gone sideways because two teams had different understandings of the same goal. You are brought in after the fact to work out what happened. What do you look for first?

| Vertex | Label | Instinct |
|---|---|---|
| V | The Language Auditor | Go back to the original brief and find the phrase that each team reasonably read differently |
| P | The Pattern Matcher | Ask each team to describe what they thought the project would produce — expect two different pictures |
| T | The Process Reviewer | Map each team's workflow and find the point where the handoff was never clearly defined |

*Why this works:* Diagnostic in framing but distinct from Scenario 1, which concerns a failing system. This scenario concerns a failing communication. Verbal looks at language, Pattern looks for the recurring structural failure class, Technical looks at process handoffs.

---

### Scenario 8 — The Unfamiliar Territory
*P · N · T*

> You are getting into something completely new — a different field, industry, or domain you have never worked in before. It could be a new job, a course, a side project, or just deep personal curiosity. What is your natural way in?

| Vertex | Label | Instinct |
|---|---|---|
| P | The Analogy Finder | Look for the underlying structure — most fields have the same dynamics underneath, dressed differently |
| N | The Metric Reader | Find out what numbers matter here — what gets measured, what gets optimized, what counts as success |
| T | The System Learner | Understand how the core tools and processes actually work — the machinery underneath |

*Why this works:* The only triangle pairing N and T directly. The broad prompt framing makes it relatable to students, career changers, and specialists, not only professional generalists.

---

## 8. Coverage and Balance

The 8-triangle battery was derived from a 10-scenario longlist. Two scenarios were dropped:

- **The Data Dump** (N · P · G) was removed because its analytical workplace context overlapped too closely with The Broken Process, and G was already overrepresented at 6 appearances.
- **The Blank Page** (V · G · C) was removed because it was an exact duplicate aptitude combination of The Vague Brief — two triangles testing identical vertex combinations add no new signal.

The resulting coverage is:

| Aptitude | Appearances | Unique Competitors Faced |
|---|---|---|
| G — Logical reasoning | 4 | 5 of 5 |
| N — Quantitative | 3 | 5 of 5 |
| V — Verbal & linguistic | 4 | 5 of 5 |
| P — Pattern recognition | 4 | 5 of 5 |
| T — Technical / hands-on | 5 | 5 of 5 |
| C — Creative thinking | 4 | 5 of 5 |

Every aptitude faces all five possible competitors across the battery. All eight aptitude combinations are unique — no two triangles test the same set of three aptitudes.

T appears five times versus three to four for other aptitudes. This is intentional: T is the most behaviourally distinctive aptitude in the set and the additional measurement reduces noise rather than introducing redundancy. N appears three times — the structural minimum — because N resists appearing in scenarios without making the quantitative angle feel contrived.

---

## 9. Interpreting the Output

The Perceived Predisposition Profile answers one question: **when confronted with an unfamiliar problem and no instruction about how to approach it, which cognitive tools does this person instinctively reach for?**

Scores should be interpreted directionally, not as fixed ceilings.

| Score | Interpretation |
|---|---|
| 4.0 – 5.0 | Strong instinctive affinity. This person consistently and naturally reaches for this cognitive approach across varied contexts. |
| 3.0 – 3.9 | Moderate affinity. This approach appears in their instincts but is not dominant. |
| 2.0 – 2.9 | Weak affinity. This approach is rarely the first instinct, though it may be used when other options are absent. |
| 1.0 – 1.9 | This cognitive approach is consistently deprioritised in favour of others. Not a deficit — simply a different natural orientation. |

The most meaningful output is not any single score but the **shape of the profile** — which aptitudes dominate relative to which others, and how consistently that pattern holds across different scenario contexts.

---

## 10. Connecting Aptitudes to Skills

The Aii is designed as an upstream component of a broader framework that maps aptitude profiles to skill development paths. Each aptitude dimension has a natural affinity with specific skill clusters:

| Aptitude | Associated Skill Clusters |
|---|---|
| G — Logical reasoning | Structured problem solving, systems thinking, problem formulation, epistemic calibration |
| N — Quantitative | Research and analysis, data interpretation, verification |
| V — Verbal & linguistic | Instruction clarity, feedback refinement, risk communication |
| P — Pattern recognition | Cross-domain synthesis, systems thinking, aesthetic judgment |
| T — Technical / hands-on | AI literacy, workflow orchestration, human-AI coordination, delegation and oversight |
| C — Creative thinking | Problem formulation, aesthetic taste, cross-domain synthesis |

A strong aptitude profile in a given dimension does not guarantee fast skill acquisition — it indicates that the learning path in that domain is likely to feel more natural, require less activation energy, and produce more durable results.

---

## 11. Intended Uses and Limitations

The Aii is designed as a low-stakes, high-engagement exploratory tool. It is most useful as a starting point for conversation, not as a definitive verdict.

**Appropriate uses:**
- Career counselling and vocational guidance — to initiate self-reflection and focus the conversation
- Learning design — to identify where a person's natural cognitive approach aligns with a skill domain
- Team development — to help people articulate their problem-solving instincts and appreciate different approaches in others
- Personal development — as one input among several in understanding how you think

**What the Aii cannot do:**
- Measure actual ability, speed, or capacity under performance conditions
- Replace validated psychometric instruments where formal assessment is required
- Account for context-dependent performance that varies by environment, stress, or motivation
- Produce results that are independent of the respondent's level of self-awareness

The output is only as good as the honesty and engagement of the person taking it. Respondents who answer strategically — based on who they want to be rather than how they actually behave — will produce a profile that reflects their aspirational self-concept rather than their instinctive cognition. This is a feature of all self-report instruments and is not unique to the Aii.

---

*Aii Methodology Document — Version 2.1*
