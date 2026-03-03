You are acting as the primary senior engineer for this codebase.

Your task is to extract and externalize the system’s long-term knowledge so that future changes do NOT require re-deriving intent from code.

This is a one-time, expensive analysis. Optimize for correctness and durability, not token efficiency.

You must produce the following artifacts:

1. ARCHITECTURE.md
   - Overall system purpose
   - High-level component diagram (textual)
   - Module boundaries and responsibilities
   - Explicit layering rules
   - Forbidden dependencies and anti-patterns
   - Design principles that explain *why* the system is structured this way

2. CONTRACTS.md
   For each major module:
   - Purpose
   - Public interfaces (conceptual, not full signatures)
   - Invariants that must never be broken
   - What the module is allowed to change internally vs externally
   - Stability expectations (stable / evolving / legacy)

3. TECHNICAL_DEBT.md
   - Known design flaws or compromises
   - Why they exist
   - Impact
   - Acceptable vs unacceptable debt
   - Priority order for addressing them

Rules:
- Do NOT refactor code yet.
- Do NOT propose improvements yet.
- Do NOT speculate beyond what can be justified from the code.
- If intent is ambiguous, document the ambiguity explicitly.
- Prefer concise, authoritative statements over narrative explanations.
- These documents must be usable as a source of truth without reading the code.

Output only the three documents, clearly separated.
