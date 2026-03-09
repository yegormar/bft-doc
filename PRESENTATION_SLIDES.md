# Built for Tomorrow: Presentation slide ideas

High-level outline for a 3-person school presentation. Each slide is bullet points only; hints for diagrams or live UI are in square brackets.

---

## Slide 1: Title

- **Built for Tomorrow**
- AI-powered strength discovery and career guidance for students and young adults
- [Optional: team names / roles]

---

## Slide 2: The problem

- Young people choose studies and careers in a market reshaped by AI
- Most guidance relies on job titles and past data, not structural change
- Personality tests say what you like, not what stays valuable
- Need: a clear picture of **potential**, **motivation**, and **skills** in an AI-era landscape

---

## Slide 3: Our premise

- Routine cognitive work becomes cheap and abundant
- Human premium shifts to designing, governing, and humanizing systems
- Choices should compound over 10-15 years, not just match today’s jobs
- [Diagram: simple “then vs now” or “value migration” sketch]

---

## Slide 4: Solution in one sentence

- Use **traits** (potential), **values** (motivation), and **skills** (trainable) to recommend skill stacks and career directions that fit an AI-shaped future
- One platform: discovery interview, then report and recommendations

---

## Slide 5: Three dimensions we measure

- **Traits:** how you tend to behave (e.g. experimentation, collaboration, ambiguity resilience)
- **Values:** what drives you (e.g. mastery, autonomy, helping others, work-life balance)
- **Skills:** what you can build (e.g. systems thinking, AI literacy, human-AI coordination)
- [Diagram: three circles or layers: traits, values, skills]

---

## Slide 6: How we assess (triangle method)

- Instead of “agree / disagree,” we use **triangles**: three options, you place yourself on the triangle
- Captures **relative preference** (trade-offs), not just yes/no
- Each corner = one dimension (trait or value); position = barycentric coordinates
- [Diagram: one triangle with three vertices and a dot; optional formula a + b + c = 1]

---

## Slide 7: Why triangles work better

- One question gives a continuous position, not a single label
- Forced trade-off reflects how values work in real life
- Same dimension can appear in different triangles for richer signal
- [Optional: short comparison table: “Traditional scale vs triangle”]

---

## Slide 8: User journey (high level)

- Short **pre-survey** (optional): who you are, context
- **Discovery:** series of triangle (or scenario) questions, one at a time
- **Results:** strength profile, traits and values scores, your answers
- **Recommendations:** skills that fit you, career directions, next steps
- [Diagram: linear flow: Pre-survey → Discovery → Results → Recommendations]

---

## Slide 9: Architecture at a glance

- **Frontend (bft-ui):** React SPA, Chakra UI, Vite
- **Backend (bft-api):** Express, in-memory sessions, optional LLM for narrative
- **Two assessment modes:** triangles (fixed question set) or scenarios (LLM-generated questions)
- [Diagram: simple 3-box: User → [UI] → [API] → [Data / LLM]; or reuse ARCHITECTURE.md diagram]

---

## Slide 10: Tech stack

- **UI:** React, React Router, Chakra UI, Nivo (radar charts), Vite
- **API:** Node.js, Express, CORS, env-based config (no hardcoded defaults)
- **Assessment data:** JSON (dimensions, triangles, skills, NOC occupations)
- **Optional:** Ollama (local or cloud) for scenario questions and report narrative

---

## Slide 11: Discovery flow (what the user sees)

- One question per screen: title, prompt, triangle or single-choice options
- Progress: “Question X of Y,” percent complete
- No account required; session id tracks progress; answers stored in session
- [Show: UI screenshot of one triangle question and progress bar]

---

## Slide 12: Results: strength profile and dimensions

- Radar or bar charts for traits and values (low / medium / high)
- Short labels so users see “where I stand” at a glance
- Optional LLM-generated narrative summary of the profile
- [Show: UI screenshot of Results profile or traits-values page with radar]

---

## Slide 13: Results: your answers

- List of questions and the user’s answers (and triangle positions if applicable)
- Supports “edit answers and recalculate” so users can play with choices
- [Show: UI screenshot of “Your answers” page]

---

## Slide 14: Recommendations: skills that fit you

- Skills ranked by fit to the user’s traits and values
- Each skill has short description and relevance to AI-era work
- User can explore and select skills to see career paths
- [Show: UI screenshot of Skills or Recommendations page]

---

## Slide 15: Careers: occupations by selected skills

- User picks skills; API returns occupations (NOC 2021) scored by match
- Grouped by category (e.g. major groups) or flat list
- Detail view: job title, duties, requirements
- [Show: UI screenshot of careers list or occupation detail]

---

## Slide 16: Design and product choices

- **No single “correct” path:** we show multiple directions and fits
- **Strength-first:** start from what the user is and wants, then recommend
- **Config-driven:** assessment mode, LLM, and prompts configurable; no defaults in code for required settings
- [Optional: 2-3 bullet “principles” graphic]

---

## Slide 17: Challenges we ran into

- Balancing **triangle coverage** (which dimensions in which triangles) with questionnaire length
- **Report caching:** when to recompute (e.g. after editing answers) and what to show without LLM
- **Two modes:** triangles (deterministic) vs scenarios (LLM); keeping both paths maintainable
- [Optional: one short “lesson learned” per person]

---

## Slide 18: Demo plan (for live presentation)

- **Person 1:** Open app, quick pre-survey (or skip), start discovery
- **Person 2:** Answer 2-3 triangle questions, show progress, complete discovery
- **Person 3:** Walk through Results (profile, traits-values, answers), then Recommendations and Careers
- [Show: live or short recorded walkthrough; keep to 3-5 minutes]

---

## Slide 19: What we would do next

- Persist sessions and answers (e.g. database) for resume and multi-device
- Richer career narratives (e.g. LLM-generated “why this fits you”)
- More triangle sets or languages
- Optional: A/B tests on question wording and report format
- [Optional: 3-4 bullet “roadmap” or “future work”]

---

## Slide 20: Conclusion and Q&A

- Built for Tomorrow: strength discovery and AI-era career guidance in one flow
- Triangles + traits/values/skills + report and recommendations
- Thank you; we’re happy to take questions
- [Optional: repeat title + team names; link or QR to repo/demo if allowed]

