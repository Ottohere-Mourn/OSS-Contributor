# Phase 5: Report Generation Prompt

## Context

You are generating a contribution report designed for **interview preparation**. The user will read this report 1-3 months from now, before a job interview, to refresh their memory and prepare answers.

You have access to all pipeline artifacts:
- `repos.json` — Phase 1 discovery
- `opportunities.md` — Phase 2 evaluation
- `selection.json` — Phase 3 human selections
- `pr-links.json` — Phase 4 execution (PR URLs, status, changes, lessons_learned)

## Core Purpose

**This is NOT a work log. It is an interview-prep document.** Every section should answer the implicit question: "If an interviewer asks me about this, can I give a smart answer?"

The report should help the user:
1. Recall what they did and WHY (not just what)
2. Demonstrate deep understanding of the code they touched
3. Anticipate and prepare for likely interview questions
4. Map their scattered contributions into a coherent "domain expertise" narrative

## Tone

Professional, honest, specific. No fluff adjectives. If there's a knowledge gap, admit it — self-awareness beats bluffing in an interview. Use bullet points and tables where they improve scanability.

## Section Requirements

### 1. Discovery at a Glance

Quick summary table. Condensed — this isn't the interesting part.

### 2. Contributions (one subsection per PR)

For each PR, the four sub-sections are:

**What I Did**: 2-3 sentences maximum. What was the problem, what was the fix.

**Codebase Context** (CRITICAL): This is where interview depth comes from. Read the files the PR touched AND 1-2 neighboring files. Describe:
- What subsystem/module does this code live in?
- How does it connect to the rest of the codebase?
- What design patterns or conventions did you observe?
- If the project has an unusual architecture choice, note it.

**Interview Q&A** (CRITICAL): Generate 3-5 realistic interview questions about this specific PR, each with a prepared answer. These should feel like real interviewer questions, not softball:

- "Why this approach over alternatives?" — show you considered trade-offs
- "What was hardest?" — show you can identify complexity honestly
- "How would you extend this?" — show forward-thinking
- "What does this codebase do differently?" — show comparative knowledge
- For bugfixes: "How did you root-cause this?" — show debugging methodology
- For docs: "What makes good documentation for this kind of project?" — show you think about developer experience

Answers should be 2-4 sentences. Concise but specific enough to use directly in conversation. If you don't have enough information to answer confidently, say so rather than making things up.

**Changes**: Simple file/change table.

### 3. Failed Attempts

If any PRs failed, include them. An honest postmortem of a failed attempt is often more impressive in an interview than a successful one — it shows you can analyze failure.

### 4. Domain Knowledge Map (CRITICAL)

This is the synthesis section — the "big picture" takeaway. An interviewer might ask: "So what did you learn about {domain} from actually contributing to it?"

- **Common Patterns**: What architectural or design patterns repeat across repos?
- **Surprising Findings**: What was counterintuitive? What did you expect that turned out wrong?
- **Technologies Demonstrated**: A checklist of specific tech/concepts touched — this is directly usable for resume bullet points.

### 5. Self-Assessment

- **Strengths**: What does this session prove you can do?
- **Gaps**: What would you need to study if asked deeper questions? Honest self-assessment is more credible than pretending to know everything.

### 6. PR Links

Simple link list.

## Edge Cases

- **Failed PRs**: Include in both Section 2 (with failure note) and Section 3. Be specific about what went wrong and what was learned.
- **No repos found**: Generate a null report with search parameters and broadening suggestions.
- **No selections made**: Report covers Phase 1-2 only, with a note that no targets were selected.
- **Insufficient data for Q&A**: If the PR was too trivial to generate meaningful answers, note "This was a straightforward {typo/docs} fix — minimal technical depth" rather than inventing fake complexity.
