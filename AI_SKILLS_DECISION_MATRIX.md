# AI-Era Skill Investment Decision Matrix  
### For High School and College Students

This framework evaluates **skills as long-term capital assets**, not professions.

At the student stage, the strategic question is not:
> “What job should I choose?”

It is:
> “What skills will compound and position me toward leverage and authority over 10–15 years?”

---

# 1. Structural Dimensions (Skill-Level)

Score each skill from **0–5** on each dimension.

| Dimension | Core Question | Why It Matters |
|------------|--------------|----------------|
| **AI Resistance** | How resistant is this skill to being performed autonomously by AI at expert level within 10–15 years? | Higher resistance reduces long-term fragility. |
| **Leverage Multiplier** | Does this skill increase your ability to use AI or coordinate others? | Some skills amplify all other skills. |
| **Authority Pathway** | Does mastering this skill position you to own decisions later? | Long-term security comes from decision ownership. |
| **Scarcity Durability** | Is this skill structurally scarce (legal, embodied, relational, physical)? | Scarcity drives premium over time. |
| **Cross-Domain Transferability** | Can this skill apply across industries? | Protects against sector disruption. |
| **Time-to-Compound** | Does early investment create exponential returns over 10+ years? | Some skills must start early to master deeply. |

---

# 2. Scoring Scale (0–5)

### AI Resistance
- **0** = Easily replicated by AI  
- **5** = AI-resistant  

### Leverage Multiplier
- **0** = Standalone execution skill  
- **5** = Amplifies many other skills  

### Authority Pathway
- **0** = Execution-only  
- **5** = Direct path to decision ownership  

### Scarcity Durability
- **0** = Abundant / easily automated  
- **5** = Structurally scarce  

### Cross-Domain Transferability
- **0** = Narrow application  
- **5** = Universal applicability  

### Time-to-Compound
- **0** = Learnable quickly anytime  
- **5** = Must start early to master  

---

# 3. Example Skill Evaluations

## Routine Coding (Implementation-Only)

| Dimension | Score |
|------------|--------|
| AI Resistance | 1 |
| Leverage Multiplier | 2 |
| Authority Pathway | 1 |
| Scarcity Durability | 1 |
| Transferability | 3 |
| Time-to-Compound | 2 |

**Interpretation:** Useful foundation, but weak long-term moat unless layered with systems thinking and architecture.

---

## Systems Thinking

| Dimension | Score |
|------------|--------|
| AI Resistance | 4 |
| Leverage Multiplier | 5 |
| Authority Pathway | 4 |
| Scarcity Durability | 4 |
| Transferability | 5 |
| Time-to-Compound | 5 |

**Interpretation:** Core compounding asset. High structural durability and leadership potential.

---

## Statistical Reasoning (Interpretation, Not Computation)

| Dimension | Score |
|------------|--------|
| AI Resistance | 3 |
| Leverage Multiplier | 5 |
| Authority Pathway | 4 |
| Scarcity Durability | 4 |
| Transferability | 5 |
| Time-to-Compound | 4 |

**Interpretation:** Critical for governance, risk ownership, and oversight roles.

---

## High-Level Communication (Persuasion & Negotiation)

| Dimension | Score |
|------------|--------|
| AI Resistance | 3 |
| Leverage Multiplier | 4 |
| Authority Pathway | 5 |
| Scarcity Durability | 5 |
| Transferability | 5 |
| Time-to-Compound | 5 |

**Interpretation:** Direct path toward influence, authority, and high-trust roles.

---

## AI Tool Fluency (Prompting + Delegation)

| Dimension | Score |
|------------|--------|
| AI Resistance | 4 |
| Leverage Multiplier | 5 |
| Authority Pathway | 3 |
| Scarcity Durability | 3 |
| Transferability | 5 |
| Time-to-Compound | 3 |

**Interpretation:** Essential baseline capability. Not sufficient alone — must stack with domain depth.

---

## Skilled Trade (e.g., Electrical Systems)

| Dimension | Score |
|------------|--------|
| AI Resistance | 4 |
| Leverage Multiplier | 3 |
| Authority Pathway | 3 |
| Scarcity Durability | 5 |
| Transferability | 3 |
| Time-to-Compound | 4 |

**Interpretation:** High durability due to physical constraint and infrastructure coupling.

---

# 4. Skill Risk Profiles

## High-Risk Skill Investment
- AI Resistance ≤ 1  
- Leverage Multiplier ≤ 2  
- Scarcity Durability ≤ 2  

Do not build long-term identity around these alone.

---

## Foundational Amplifier Skill
- Leverage Multiplier ≥ 4  
- Transferability ≥ 4  
- AI Resistance ≥ 3  

Prioritize early and deeply.

---

## Authority-Track Skill
- Authority Pathway ≥ 4  
- Scarcity Durability ≥ 4  
- Leverage Multiplier ≥ 3  

These build long-term decision power and income resilience.

---

# 5. Skill Portfolio Strategy

Instead of choosing one skill, build a **stack**:

## Layer 1 – Amplifier Core
- Systems thinking  
- Statistical reasoning  
- Communication  
- AI literacy  

## Layer 2 – Domain Anchor
Choose one:
- Engineering  
- Law + technology  
- Healthcare + data  
- Energy systems  
- Finance + risk  
- Robotics / advanced manufacturing  

## Layer 3 – Authority Development
- Decision ownership exposure  
- Negotiation  
- Stakeholder management  
- Ethical judgment  
- Risk assessment  

---

# 6. Strategic Insight

The durable premium will not favor:

> “I execute tasks efficiently.”

It will favor:

> “I define problems, coordinate systems, and own consequences.”

At the student stage, you are not choosing a job title.

You are choosing:
- What compounds  
- What creates leverage  
- What positions you toward authority instead of execution  

---

# 7. Implementation Note

The app turns structural scores into a single "AI future relevance" (0-1) score using a configurable model so it can be tuned without code changes. That model is in **bft-api/src/data/ai_skills_ranking_model.json**: it lists the six dimensions (all "higher = more beneficial") and how they are combined (e.g. average). Edit that JSON to change weights, add dimensions, or adjust the ai_trend fallback values.