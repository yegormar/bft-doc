# About Page

Canonical description of the About page for Built for Tomorrow. Use this to keep the UI and copy aligned.

## Purpose

- Explain what Built for Tomorrow is and why it exists.
- Build trust (privacy, no sign-up, strengths-based).
- Drive action: Home or Start discovery.

## Audience

- Teenagers and young adults.
- Often on mobile; content must be short and scannable.

## Sections (in order)

1. **Hero**
   - Title: "About Built for Tomorrow"
   - Tagline: "Your strengths, your future"
   - One-line value prop: "Explore careers with AI. No sign-up. No tracking."
   - Primary CTA: "Start discovery" button above the fold (links to /discovery). Aria-label: "Start your discovery journey".

2. **Our Mission**
   - One short paragraph: "We help you answer 'What do I do with my life?' by mapping your strengths, values, and interests to real options in a world shaped by AI."
   - Four pillars (icons + title + short description): For you, Strengths-based, Private, AI-powered. Copy is short and direct (e.g. "We focus on what you're good at, not fixed career boxes.").

3. **AI-Era Labor**
   - One short paragraph: "As AI changes work, some skills matter more: thinking in systems, making decisions, adapting. Routine digital tasks matter less. Your unique strengths matter more."

4. **Our Approach**
   - One short paragraph: self-reflection and AI-powered questions; we don't put you in one job box; we show how you think and solve problems so you can adapt as work changes.

5. **Meet the Team**
   - Heading: "Meet the Team" (not "Meet the Developers"). Icon: people/team (e.g. UsersRound), not code.
   - Each person: headshot and name only. No role, no bio. Images: loading="lazy", decoding="async"; alt="" (name in heading).

6. **Footer**
   - Two buttons only: Home, Start discovery. No status line, no "For: Students & young adults", no testimonial quote.

## Constraints

- **No Technology section.** Do not list stack (React, Chakra UI, etc.) on the About page.
- **No long dashes.** In all project files (including this doc and the About page copy), use a hyphen (-), comma, or rephrase instead of em dash (—) or en dash (–).
- **Mobile-first.** Tighter spacing and smaller type on small viewports; minimal scrolling.
- **Short copy.** Every block should be scannable in a few seconds.

## Data and tests

- Team list (names, headshots) can stay in the page component or a small data file; roles/bios may be kept in data but are not displayed.
- Test IDs: `page-about`, `about-hero-title`, `about-hero-tagline`, `about-hero-cta`, `about-mission-title`, `about-thesis-title`, `about-approach-title`, `about-link-home`, `about-link-start`.
- Sections use `<section>` with `aria-labelledby` pointing to the section heading id for screen reader navigation.
