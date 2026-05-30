# Phase 5: Report Generation Prompt

## Context

You are generating the final contribution report. You have access to all artifacts from the pipeline:

- `repos.json` — Phase 1 discovery: all candidate repos with scores
- `opportunities.md` — Phase 2 evaluation: detailed opportunity analysis
- `selection.json` — Phase 3 human selections: which targets were chosen
- `pr-links.json` — Phase 4 execution: PR URLs, status, changes

## Report Requirements

### Audience

The reader is the user themselves, 1-3 months from now, preparing for a job interview. They need to quickly recall:
1. What they explored and why
2. What they chose to work on and why
3. What each PR accomplished, technically
4. What they learned from the process

### Tone

Professional, concise, data-backed. This is not a blog post — it's a technical retrospective. Use bullet points and tables where they improve scanability. No fluff adjectives ("amazing", "incredible").

### Structure

Generate the report following the template in `templates/report-template.md`. Key sections:

1. **Header**: date, domain, parameters
2. **Discovery Summary**: one table showing all evaluated repos, their scores, and the final decision
3. **Contributions Made**: one subsection per PR, each with:
   - Metadata row (repo, issue, type, difficulty, PR link, status)
   - Technical Approach (2-3 sentences on WHY, not just WHAT)
   - Changes Made (bullet list of files + descriptions)
   - Lessons Learned (1-2 sentences, be honest — was it harder than expected? Any surprises?)
4. **Key Insights**: cross-cutting observations about the contribution experience

### Data Source Priority

When populating the report:
- Use `pr-links.json` for final PR URLs and status — this is the ground truth
- Use `selection.json` for which targets were chosen
- Use `opportunities.md` for the pre-execution assessments (difficulty, impact estimates)
- Use `repos.json` for the discovery summary table

### Edge Cases

- **If a PR failed**: include it in the report with a clear "FAILED" status and the failure reason. Be honest — failed attempts are learning material too.
- **If no repos found**: generate a "Null Report" with the search parameters and suggestions for broadening the search.
- **If human selected nothing**: the report should only have Phase 1-2 data, with a note that no targets were selected.
