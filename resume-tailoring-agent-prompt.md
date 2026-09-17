You are an expert technical resume strategist, career-gap analyst, and meticulous LaTeX editor. Your task is to create an aspirational **fictional ideal-candidate model** for the job description below. Starting from my current experience and projects, rewrite the content of my resume to show what my background would ideally look like if I had deliberately developed it into a complete match for this role.

This output is a private career-development artifact, **not an application resume**. It is intentionally hypothetical and must never be represented as my real background or submitted to employers. Its value is in showing me exactly which experience, projects, outcomes, and skills I should work toward. Despite being fictional, it must be optimized as rigorously as a real ATS-ready application resume, so it also teaches me the content, terminology, structure, and keyword placement of a strong submission.

## Inputs

- Current-experience foundation: `resume-recall.md`
  - This file describes my real work, projects, technologies, outcomes, metrics, constraints, and interview-ready context.
  - Use it to understand my current starting point, recurring strengths, plausible technical trajectory, and the work already adjacent to the target role.
- Resume to tailor: `resume.tex`
- Target job description:

<PASTE THE COMPLETE JOB DESCRIPTION HERE>

## Non-negotiable scope and file rules

1. Edit **only** `resume.tex`. Do not create, modify, delete, rename, compile, or regenerate any other file. In particular, do not touch PDFs, images, `.aux`, `.log`, `.out`, `.fls`, `.fdb_latexmk`, lock files, README files, or `resume-recall.md`.
2. Preserve the existing LaTeX preamble, custom commands, contact information, education facts, section order, and overall template/layout unless a minimal change inside `resume.tex` is essential to fit the tailored content.
3. Do not change employers, job titles, dates, education, contact details, links, or section structure. The fictionalization must occur in experience/project bullet points, project technology labels, and Technical Skills only.
4. Make the changes directly in `resume.tex`; do not merely propose them in prose.

## Fictional-model standard

Create a resume that reads as though I became an excellent match for this role by extending and deepening my real trajectory. It should be credible and technically coherent, but it is explicitly allowed to contain fictional achievements, skills, projects, technical scope, and metrics needed to model the target profile.

- Use `resume-recall.md` as the seed material. Preserve the recognizable themes of my background—such as backend/full-stack development, performance, caching, distributed systems, cloud, CI/CD, and measurable operational impact—then evolve them toward the JD.
- Treat all additions or modifications that exceed my current evidence as intentional targets: realistic work I would need to perform, learn, and be able to defend in order to earn the role.
- Make the hypothetical progression plausible. Extend adjacent capabilities before making large leaps, and write technically detailed claims that could be turned into a concrete learning/project plan.
- You may add JD-required skills, tools, responsibilities, domains, metrics, and project details even when absent from the evidence bank, provided they make sense for the target role and are integrated into believable bullets.
- Do not change immutable identity/history facts: employer names, titles, dates, degree, contact information, or links. Do not add certifications, awards, employers, roles, education, or whole new sections.
- Do not use obvious placeholders, disclaimers, brackets, or labels such as “fictional,” “aspirational,” or “to learn” inside `resume.tex`; it should read like a polished target-state resume. The final response will make the private-only status clear.

## Tailoring objective

Optimize `resume.tex` for a recruiter, hiring manager, and applicant tracking system reviewing this exact job description. The resulting document should demonstrate the ideal target state: a complete, compelling match for the role built on a plausible evolution of my actual foundation. Treat ATS optimization as a first-class goal: use standard headings, conventional job-title and skill names, JD terminology, role-relevant keyword coverage, and readable accomplishment bullets while maintaining a clean one-page-oriented LaTeX format.

1. Read the entire job description and extract:
   - target title/seniority and likely hiring priorities;
   - required versus preferred skills;
   - core languages, frameworks, cloud/platform tools, databases, development practices, and business/domain terms;
   - expected ownership, collaboration, reliability, performance, testing, security, CI/CD, and system-design signals;
   - important repeated keywords and exact terminology worth using naturally.
2. Read all of `resume-recall.md` and inspect the existing `resume.tex`.
3. Build an internal mapping of (a) JD requirements already adjacent to my evidence and (b) new capabilities, achievements, or projects that would close each remaining gap.
4. Rewrite and reorder content in `resume.tex` to maximize target-state alignment:
   - Prioritize the experience bullets most relevant to the role. Reorder bullets within an employer when helpful, without altering chronology, employer names, or titles.
   - Replace generic or lower-value bullets with stronger target-state bullets. Reuse and sharpen real achievements where relevant; invent plausible, role-relevant achievements where doing so models an important missing requirement.
   - Tailor project bullets and project technology labels to demonstrate the JD's core requirements. Use existing projects as the base; model the features, architecture, quality practices, and deployment scope they would need.
   - Tune the Technical Skills section to present the full, relevant target stack in a clean hierarchy. Include every important JD skill that a genuinely ideal candidate should have.
   - Use the JD's exact terminology naturally in bullets and skills; do not keyword-stuff.
   - Preserve useful real quantified impacts when aligned, and create realistic, internally consistent target metrics where a fictional bullet needs one. Never make metrics so extreme that the target profile feels implausible.
5. Keep the resume concise, credible, and one-page oriented. Aim for roughly the current density. Favor high-signal accomplishments over exhaustive coverage. Remove redundancy before making the document longer.

## Writing requirements

- Write accomplishment-focused bullets in active voice: strong action + relevant technical approach + measurable/resulting impact.
- Make each bullet independently understandable. Include implementation detail only when it proves relevance or technical depth.
- Use precise technical wording detailed enough to become an interview-preparation and learning target.
- For software roles, surface relevant systems thinking: design, APIs, distributed systems, data, reliability, performance, testing, CI/CD, observability, security, and collaboration when the JD calls for them.
- Avoid vague filler such as “responsible for,” “worked on,” “helped,” “various,” “utilized,” “hardworking,” and unsupported adjectives such as “expert,” “world-class,” or “highly scalable.”
- Maintain consistent tense, punctuation, capitalization, and LaTeX escaping. Keep all valid existing macros and environment structure intact.
- Preserve ATS readability: conventional headings, plain skill names, standard job titles, no tables added for body content, no graphics, no columns, and no keyword dump.
- Optimize for semantic ATS matching, not just keyword frequency: place important JD terms in context within the most relevant experience/project bullets and Technical Skills, use common spelling variants only when natural, and make the target title, core stack, scope, and measurable outcomes easy to parse.

## Quality-control checklist

Before finishing, review the changed `resume.tex` and confirm internally that:

- every JD-critical requirement is either represented through a polished target-state bullet/skill or consciously excluded only because it is irrelevant to the role;
- the fictional additions form a plausible evolution of the starting experience in `resume-recall.md`;
- JD-critical keywords appear naturally in the most relevant experience/project bullets and/or Technical Skills;
- the document uses conventional ATS-friendly headings and plain, parsable terminology without keyword stuffing, graphics, or layout changes that obscure content;
- facts that must not change (name, contact information, employers, titles, dates, degree) remain accurate;
- the LaTeX syntax and existing command structure remain valid;
- the result is concise and does not materially exceed the current one-page-oriented layout;
- only `resume.tex` has been modified.

## Final response

After editing, provide a brief report containing:

1. A one-sentence statement that only `resume.tex` was changed and that it is a fictional, private target-state resume—not suitable for applications.
2. The JD themes and keywords that were emphasized.
3. A concise list of the most meaningful fictional changes.
4. A brief ATS rationale explaining how the most important JD requirements were surfaced for parsing and relevance.
5. A “development roadmap” list mapping each invented or materially expanded claim to the concrete skills, projects, responsibilities, and measurable outcomes I would need to build before I could truthfully put that claim on a real resume.

Do not ask clarifying questions. Make the best target-state modeling decision from the provided job description and repository files.
