# Home Page

Canonical description of the Home page for Built for Tomorrow. Keeps the UI and copy aligned.

## Purpose

- Land visitors (teens, young adults) with a clear value prop and reassurance.
- Explain the "shift" (reframe) the tool helps with.
- Build trust (privacy, no sign-up).
- Drive action: Start discovery (primary and secondary CTA).

## Audience

- Teenagers and young adults, often on mobile. Content is short and scannable; spacing and type are responsive.

## Sections (in order)

1. **Hero**
   - Title: "Built for Tomorrow"
   - Tagline: "Find your strengths. See where they take you."
   - One short paragraph: "Discover your strengths, explore professions and skills, and see where to invest your time. No sign-up. No one right path."
   - Primary CTA: "Start discovery" (links to /discovery). Aria-label: "Start your discovery journey".

2. **The shift you can make**
   - Four reframe cards: From (old question) / To (new question). E.g. "What job should I choose?" to "What capabilities should I build?" (aligns with Careers: skills investment, dream job, AI-safe direction); "What if AI replaces me?" to "How do I stay useful as work changes?"
   - Secondary CTA below: "Start discovery".

3. **Your privacy matters**
   - Two buttons (outline, gray), same style as About page footer: "No sign-up" (CircleCheck), "Private and free" (ShieldCheck). Not links; cursor default. One line below: "No account required. We don't sell your data."
   - Test IDs: home-trust-item-no-sign-up, home-trust-item-private-and-free.

4. **Feedback & support**
   - Heading "Feedback & support" is a link to /feedback. Test ID: home-feedback-link.
   - Text: "Built for Tomorrow is free. Got a bug, an idea, or feedback? We're listening."

## Constraints

- **No long dashes.** Use hyphen (-), comma, or rephrase; no em dash (—) or en dash (–).
- **Mobile-first.** Responsive hero and section padding; responsive heading and body font sizes (smaller on base).
- **Section semantics.** Each major block is a `<section>` with `aria-labelledby` pointing to the section heading id. Reframe list and trust list use role="list" / role="listitem".

## Test IDs

- `page-home`
- `home-hero-title`
- `home-hero-tagline`
- `home-start-discovery` (hero CTA)
- `home-reframe-title`
- `home-reframe-row` (each reframe card)
- `home-start-discovery-secondary`
- `home-privacy-title`
- `home-trust-item-no-sign-up`, `home-trust-item-private-and-free`
- `home-support-title`
- `home-feedback-link` (link to /feedback)
