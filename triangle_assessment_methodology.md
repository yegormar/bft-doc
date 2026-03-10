# Triangle Assessment Method
## A New Approach to Measuring Values and Traits

---

## The Problem With Traditional Questions

Most psychometric assessments ask people to respond to statements like:

> *"I prefer working in a team." — Strongly Agree / Agree / Neutral / Disagree / Strongly Disagree*

Or they present scenarios with labelled options:

> *"You have two job offers. Job A pays more. Job B gives you more freedom. Which do you choose?"*

Both formats share the same flaw: **the person can see what is being measured**. A perceptive teenager reads "I prefer working in a team" and immediately knows the test is measuring teamwork preference. They answer what they think sounds good, not what is actually true.

Scenario questions are better, but they still require the respondent to choose between discrete options — which forces a binary where the real answer is often somewhere in between.

---

## The Core Idea

Instead of asking a question, present a triangle.

Each corner of the triangle represents a distinct human value or behavioral tendency, described in plain, concrete language. The person drags a ball to the position on the triangle that feels most like them.

```
        [Mastery & Growth]
              /\
             /  \
            /    \
           /  ●   \
          /________\
[Helping Others]  [Recognition]
```

The ball's position is captured as three coordinates that always sum to 1.0 — a mathematical representation called **barycentric coordinates**. A ball sitting close to the Mastery corner might produce coordinates of (0.65, 0.20, 0.15), meaning the person is 65% pulled toward mastery, 20% toward helping others, and 15% toward recognition.

This single placement generates a score on all three dimensions simultaneously.

---

## Why This Works Better

### 1. Nothing to Game

The optimal position in a triangle is not obvious. There is no "right answer" visible from the question. A person cannot easily reverse-engineer a placement to perform a specific profile because the three dimensions interact in a continuous two-dimensional space rather than presenting discrete labelled options.

Contrast this with a scenario question where one option often reads as more mature or socially desirable than the others. The triangle removes that signal entirely.

### 2. Forces Real Trade-offs

The three vertices of each triangle are chosen specifically because they **genuinely compete**. Moving toward one vertex means moving away from the others. This constraint forces the person to reveal what actually matters most, rather than rating everything highly.

A Likert scale lets someone say they strongly value recognition, mastery, *and* helping others simultaneously. A triangle does not. If you move toward recognition, you drift away from the other two. The position is honest because it is physically constrained.

### 3. Captures Mixed Profiles

Traditional questions produce a single answer. A triangle produces a continuous position that captures nuance. Someone who is 45% pulled toward financial success, 35% toward work-life balance, and 20% toward belonging is a meaningfully different profile from someone who is 70/15/15 — and that difference is captured automatically without needing additional questions.

### 4. Loss and Gain Framing

The same dimensions can be measured from two different emotional angles:

- **Gain framing:** *"What pulls you toward an opportunity?"*
- **Loss framing:** *"What would make you turn it down?"*

People who are strongly driven by a value often find its absence more painful than its presence is pleasurable. Measuring the same dimension in both frames — once as attraction, once as dealbreaker — produces more reliable scores than asking the same question twice.

### 5. Engagement

For a teenager, dragging a ball on a triangle feels like an interaction, not a test. The tactile, spatial nature of the input reduces the feeling of being assessed and increases honest response. The interface invites curiosity rather than performance.

---

## How Scores Are Calculated

Each vertex is assigned a dimension. When the person places the ball, the barycentric coordinates map directly to dimension scores.

The coordinate for each vertex ranges from 0.0 (ball at the opposite edge, furthest from this vertex) to 1.0 (ball exactly at this vertex). This is converted to the 1–5 scoring scale using:

```
score = (coordinate × 4) + 1
```

| Coordinate | Score | Meaning |
|---|---|---|
| 0.0 | 1.0 | No affinity with this dimension |
| 0.25 | 2.0 | Low affinity |
| 0.33 | 2.33 | Neutral (centre of triangle) |
| 0.5 | 3.0 | Moderate affinity |
| 0.75 | 4.0 | High affinity |
| 1.0 | 5.0 | Full affinity |

When a dimension appears in more than one triangle, scores are averaged across all placements to produce a final dimension score. Dimensions measured with both gain and loss framing are averaged across both.

Interpreting a single placement as three separate scores (e.g. "A = medium 2.33, B = medium 2.33, C = medium 2.33") is misleading. The position on the triangle carries zone-based meaning: where the ball sits (edge, near-edge, centre, near-corner, corner) and whether a vertex is explicitly rejected matter as much as the raw numbers. The following section describes how placements should be interpreted.

---

## Interpreting Placements: Zones and the Rejection Signal

The ball's position is not just three coordinates. It encodes **where** you placed the ball (the zone) and **which vertex you excluded** (rejection). Treating a low coordinate as "a small preference" is wrong: a near-zero weight is an explicit rejection (that dimension is not part of your thinking), not a weak score.

### Edge Zone: Ball on the line between two vertices

Example: Ball halfway between mastery and recognition, with collaboration near zero.

You are saying three things simultaneously:

"I genuinely blend these two" - both mastery and recognition feel true
"The balance between them matters but I can't separate them" - they're both activated
"Collaboration is simply off the table" - not a weak preference, a closed door

The model must treat the third as its own piece of evidence: an explicit rejection, not just a low score.

### Near-Edge Zone: Ball close to an edge but not perfectly on it

Example: Ball at (0.55, 0.38, 0.07), somewhere between A and B but drifting toward A.

You are saying four things:

"A is my primary instinct" - it gets the most weight
"B is genuinely in play" - not dismissed, clearly present
"A and B feel connected in this context" - you couldn't fully separate them
"C is not part of my thinking at all" - the 0.07 is not a weak preference, it's an absence

The near-edge placement tells the model that C is excluded even though you were not fully committed to either A or B alone.

### Centre Zone: Ball left near the middle

Example: Ball at (0.38, 0.35, 0.27), roughly central, slightly toward A.

You are saying something much weaker:

"All three feel somewhat relevant" - no strong pull in any direction
"A edges it slightly" - but barely
"Nothing here is excluded" - even C at 0.27 is still in the picture

The centre is the least informative placement. No vertex is being rejected. The model treats this as weak evidence across all three (low zone concentration). You are not telling the model much.

### Near-Corner Zone: Ball close to a vertex but not quite there

Example: Ball at (0.65, 0.28, 0.07), clearly toward A but not fully committed.

You are saying:

"A is my dominant instinct" - clearly, but with some hesitation
"B is present but secondary" - you didn't reject it completely
"C is rejected" - the 0.07 is an exclusion, not a preference

The tension between A and B is informative: you moved toward A strongly but left meaningful weight on B (A and B may feel connected, or you were pulled between them before A won). C is excluded just as clearly as in a full corner placement.

### Summary: Rejection vs weak preference

| Situation | Wrong interpretation | Right interpretation |
|-----------|----------------------|----------------------|
| One vertex under 0.15 weight | "Low affinity for that dimension" | Explicit rejection: that dimension is not in play |
| Ball on edge between A and B | "A and B both medium" | A and B blended; the third vertex is excluded |
| Ball near centre | "All three medium" | Weak evidence; nothing excluded; low information |

Scoring and reporting (including "How this was interpreted" on the results page) must use zone-based language and call out excluded vertices, instead of reducing every placement to three band labels.

---

## Triangle Design Principles

Not every combination of three dimensions makes a valid triangle. A well-designed triangle satisfies three conditions:

**Genuine tension at the extremes.** Someone fully at vertex A should naturally score low on B and C. If two vertices represent similar things, the triangle collapses — the ball clusters in one region and produces little information.

**Interpretable without labels.** Each vertex must be described in plain, concrete language that a teenager immediately understands. Abstract dimension names like "autonomy" or "mastery" never appear — instead, the vertex says *"Nobody tells me how to do it — I own the process completely."*

**Covering a real space of human variation.** The three vertices should represent three genuinely different types of people who all exist in the real world. If you cannot picture a person fully at each vertex, the triangle is not well-designed.

---

## Applicability

### Strong fit: Values measurement

Values are what a person is optimising for — their destination. They are inherently comparative: what matters *more* to you when you cannot have everything. This is exactly what the triangle format is designed to reveal. The triangle's forced trade-off mirrors how values actually operate in real life.

Values measured well in this format include financial success vs work-life balance vs belonging, mastery vs recognition vs helping others, and autonomy vs security vs creative expression.

### Good fit: Behavioral traits

Traits describe how a person operates — their natural tendencies under pressure, in teams, or when things are uncertain. They can be measured in triangles when the three traits represent genuinely different behavioral orientations rather than different intensities of the same tendency.

Example: reading others and adjusting (social collaboration) vs starting things before they are fully defined (experimentation) vs owning outcomes regardless of who executed (comfort with ambiguous ownership). These are three distinct orientations that a person can be positioned between.

### Poor fit: Aptitudes

Aptitudes are capacities — how quickly someone can learn a domain, how naturally they reason logically, how fluently they work with numbers. Capacities cannot be measured by asking where someone places themselves on a triangle. A person who places themselves far from the logical reasoning vertex is reporting a preference or self-perception, not demonstrating their actual reasoning capacity.

Aptitudes require performance tasks or demonstrated behavior to measure accurately. The triangle format should not be used for aptitude assessment.

---

## Limitations

**Self-report bias persists.** The triangle reduces gaming but does not eliminate it. A highly self-aware person who wants to appear a certain way can still place the ball strategically. The format raises the bar for gaming but does not remove the possibility.

**Centre placements are ambiguous.** A ball placed exactly in the centre could mean genuine balance across all three dimensions, or it could mean the person found none of the vertices compelling, or it could mean uncertainty or disengagement. Centre placements should be flagged in the scoring logic and treated with lower confidence than corner-biased placements.

**Vertex wording is load-bearing.** The entire validity of the measurement depends on how the vertices are written. Wording that implies one vertex is more virtuous than another will bias placements toward that vertex regardless of the person's actual values. Every vertex must feel like an equally legitimate and attractive position.

**Dimensions that correlate cannot share a triangle.** If two dimensions naturally travel together — for example, mastery and challenge achievement both pull toward difficulty and self-improvement — placing them on the same triangle produces a degenerate shape where the ball cannot easily distinguish between them. Correlated dimensions must appear in separate triangles or be measured independently.

---

## Summary

| Property | Traditional Likert | Scenario Question | Triangle |
|---|---|---|---|
| Resistant to gaming | Low | Medium | High |
| Captures mixed profiles | No | No | Yes |
| Forces real trade-offs | No | Partial | Yes |
| Generates multiple scores per interaction | No | No | Yes |
| Works for aptitudes | Poorly | Better | No |
| Works for values | Yes | Yes | Best |
| Works for traits | Yes | Best | Good |
| Engagement for teenagers | Low | Medium | High |

The triangle format is not a replacement for all assessment methods. It is a purpose-built instrument for measuring values and behavioral traits in contexts where self-report validity matters, gaming is a real concern, and the person being assessed is old enough to engage with a spatial, interactive interface but young enough that traditional survey formats feel like homework.

# Other expert assessment
## Methodology Validation: Triangle Assessment Framework

## 1. Executive Summary
The **Triangle Assessment Framework** is a high-fidelity psychometric model designed to measure human values and behavioral traits through spatial trade-offs. By utilizing **barycentric coordinates** ($a + b + c = 1.0$), it effectively captures the finite nature of human motivation—moving toward one priority necessitates moving away from others.

## 2. Validation Scoring Matrix

| Category | Score | Rationale |
| :--- | :---: | :--- |
| **Mathematical Logic** | 10/10 | The use of barycentric coordinates is the ideal representation for zero-sum trade-offs. |
| **Bias Mitigation** | 9/10 | Replaces obvious "right/wrong" Likert scales with competing virtues, making the test harder to "game". |
| **User Engagement** | 10/10 | The tactile "drag-and-drop" interface is significantly more engaging for the target teenage demographic than traditional forms. |
| **Construct Integrity** | 8/10 | Most triangles maintain high tension. Some blended vertices (e.g., Triangle 08) may require tie-breakers for maximum precision. |

**Final Methodology Grade: 9/10**

---

## 3. Core Strengths

### A. The "Zero-Footprint" Gaming Defense
Traditional questions like *"I prefer working in a team"* are easily reverse-engineered. This methodology forces users to choose between three equally "virtuous" descriptions, such as **Mastery**, **Helping Others**, and **Recognition**, which obscures the "correct" answer and reveals true intent.

### B. Gain vs. Loss Multi-Dimensionality
The methodology's use of **framing variants** is a standout feature:
* **Gain Framing:** Measures what pulls a user toward an opportunity.
* **Loss Framing:** Measures "dealbreakers" or what a user would sacrifice to protect.
* **Result:** This dual-angle measurement captures users who may be more motivated by avoiding dissatisfaction than achieving satisfaction.

### C. AI-Era Future-Proofing
The traits measured (e.g., **Ambiguity Resilience**, **Comfort with Ambiguous Ownership**, and **Adaptability**) are specifically aligned with the skills required to thrive in an AI-augmented workforce.

---

## 4. Areas for Refinement

* **Center-Point Logic:** A ball placed exactly in the center can represent a perfect balance or a lack of engagement. Future iterations should flag "Center Placements" for lower confidence scores.
* **Vertex Isolation:** Ensure that vertices do not correlate. For example, if "Mastery" and "Achievement" are too similar, the triangle "collapses" and provides less data.
* **Aptitude Guardrail:** The methodology correctly identifies that it should not be used for aptitudes (e.g., math skills), as these require performance-based testing rather than self-reported preference.

---

## 5. Conclusion
This methodology is a robust, modern alternative to standard personality assessments. It is particularly well-suited for career and study guidance where understanding a person's "internal compass" is more important than measuring their current skill level.