You are an expert technical resume strategist, career-gap analyst, and meticulous LaTeX editor. Your task is to create an aspirational **fictional ideal-candidate model** for the job description below, but it must remain visibly derived from my current experience. Starting from my current experience and projects, reshape the content of my resume to show how I could deliberately develop each existing accomplishment into a stronger match for this role.

This output is a private career-development artifact, **not an application resume**. It is intentionally hypothetical and must never be represented as my real background or submitted to employers. Its value is in showing me exactly how to extend experience I already recognize—not replacing it with an unrelated idealized history. Despite being fictional, it must be optimized as rigorously as a real ATS-ready application resume, so it also teaches me the content, terminology, structure, and keyword placement of a strong submission.

## Inputs

- Current-experience foundation: `resume-recall.md`
  - This file describes my real work, projects, technologies, outcomes, metrics, constraints, and interview-ready context.
  - Use it to understand my current starting point, recurring strengths, plausible technical trajectory, and the work already adjacent to the target role.
- Resume to tailor: `resume.tex`
- Target job description:

<PASTE THE COMPLETE JOB DESCRIPTION HERE>

## Non-negotiable scope and file rules

1. Make content edits **only** to `resume.tex`. You must also compile it locally with the already-installed TeX Live toolchain to generate and inspect `resume.pdf`. Compilation-generated files (`resume.pdf`, `.aux`, `.log`, `.out`, `.fls`, `.fdb_latexmk`, and similar build artifacts) are permitted only as build output; do not manually edit them or modify any other source file. Do not create, modify, delete, or rename README files, images, `resume-recall.md`, or other non-build files.
2. Preserve the existing LaTeX preamble, custom commands, contact information, education facts, section order, and overall template/layout unless a minimal change inside `resume.tex` is essential to keep the tailored resume to **one page**.
3. Do not change employers, job titles, dates, education, contact details, links, or section structure. The fictionalization must occur in experience/project bullet points, project technology labels, and Technical Skills only.
4. Make the changes directly in `resume.tex`; do not merely propose them in prose.

## Fictional-model standard

Create a resume that reads as though I became an excellent match for this role by extending and deepening my real trajectory. It should be credible and technically coherent, but every changed bullet must be a recognizable evolution of a specific recall bullet or existing project—not a replacement with a different story.

- Use `resume-recall.md` as the seed material. Preserve the recognizable themes of my background—such as backend/full-stack development, performance, caching, distributed systems, cloud, CI/CD, and measurable operational impact—then evolve them toward the JD.
- Assign every rewritten experience or project bullet one **primary source bullet** from `resume-recall.md` (or, for a project, the matching existing project). Keep its core situation, product/domain, main technical action, and outcome intact unless the JD makes a small wording change necessary. A reader who knows the source should be able to recognize it immediately.
- Build each new bullet in this order: **base achievement → adjacent JD-relevant extension → resulting impact**. Retain at least one distinctive source detail in the final bullet: an original metric, technology, scale, constraint, system behavior, or outcome. Do not swap the original problem for a different one just to insert a JD keyword.
- Treat additions that exceed current evidence as small, explicit career-development targets: realistic work I would need to perform, learn, and be able to defend. Prefer adding one adjacent capability (for example, testing, observability, API design, deployment, security, or collaboration) to an existing accomplishment over inventing a new domain, system, or business result.
- Do not invent a new employer responsibility, customer domain, architecture, project purpose, or metric merely to cover a JD requirement. If a JD requirement has no honest adjacent base in the recall document, leave it out of the resume and place it in the development roadmap instead. A narrower, traceable model is more useful than artificial 100% keyword coverage.
- Make the hypothetical progression plausible. Extend adjacent capabilities before making large leaps, and write technically detailed claims that could be turned into a concrete learning/project plan.
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
3. Build an internal mapping of (a) each existing recall bullet/project to its closest JD requirements, (b) the invariant facts that must survive the rewrite, and (c) only the smallest adjacent capability that would strengthen that bullet. Put JD requirements with no credible source into the development roadmap; do not force them into `resume.tex`.
4. Rewrite and reorder content in `resume.tex` to maximize target-state alignment:
   - Prioritize the experience bullets most relevant to the role. Reorder bullets within an employer when helpful, without altering chronology, employer names, or titles.
   - Replace generic or lower-value bullets with stronger target-state bullets only when they retain a clear source accomplishment. Reuse and sharpen real achievements first; add a narrowly adjacent, plausible extension only when it explains how that same work could meet the JD.
   - Tailor project bullets and project technology labels to demonstrate the JD's core requirements only through existing projects. Preserve each project's purpose and core stack; model adjacent features, architecture, quality practices, or deployment scope it would need rather than turning it into a different project.
   - Tune the Technical Skills section to present the relevant target stack in a clean hierarchy, but add a JD skill only when it is supported by the current evidence or directly tied to an adjacent expansion in a rewritten bullet. Put unsupported JD skills in the roadmap instead of presenting them as resume experience.
   - Use the JD's exact terminology naturally in bullets and skills; do not keyword-stuff.
   - Preserve useful real quantified impacts when aligned, and create realistic, internally consistent target metrics where a fictional bullet needs one. Never make metrics so extreme that the target profile feels implausible.
5. The finished resume must fit on **exactly one page** when compiled locally with the already-installed TeX Live setup. This is a hard requirement, not a preference. After every material content change, run the local LaTeX build to generate `resume.pdf` and inspect the PDF page count. If it is two or more pages, revise `resume.tex` to remove or tighten lower-value content, rebuild, and repeat until the generated PDF is exactly one page. Favor high-signal accomplishments over exhaustive coverage; do not shrink fonts, margins, spacing, or readability merely to force content onto one page.

## Writing requirements

- Write accomplishment-focused bullets in active voice: strong action + relevant technical approach + measurable/resulting impact.
- Preserve the causal chain of the source bullet: the original problem/constraint, the central implementation, and the original result must remain recognizable. Rephrase for the JD; do not replace the chain with an unrelated accomplishment.
- Keep source metrics accurate. New metrics are allowed only when they are a direct, conservative extension of the same system and can be explained in the final roadmap. Never replace a real metric with a fabricated one solely because it sounds more aligned.
- Make each bullet independently understandable. Include implementation detail only when it proves relevance or technical depth.
- Use precise technical wording detailed enough to become an interview-preparation and learning target.
- For software roles, surface relevant systems thinking: design, APIs, distributed systems, data, reliability, performance, testing, CI/CD, observability, security, and collaboration when the JD calls for them.
- Avoid vague filler such as “responsible for,” “worked on,” “helped,” “various,” “utilized,” “hardworking,” and unsupported adjectives such as “expert,” “world-class,” or “highly scalable.”
- Maintain consistent tense, punctuation, capitalization, and LaTeX escaping. Keep all valid existing macros and environment structure intact.
- Preserve ATS readability: conventional headings, plain skill names, standard job titles, no tables added for body content, no graphics, no columns, and no keyword dump.
- Optimize for semantic ATS matching, not just keyword frequency: place important JD terms in context within the most relevant experience/project bullets and Technical Skills, use common spelling variants only when natural, and make the target title, core stack, scope, and measurable outcomes easy to parse.

## Quality-control checklist

Before finishing, review the changed `resume.tex` and confirm internally that:

- every JD-critical requirement is either represented through a polished, traceable target-state bullet/skill or listed in the development roadmap because it lacks a credible source;
- every changed bullet has a specific recall/project source and retains a distinctive source detail and causal chain;
- the fictional additions are the smallest plausible evolution of the starting experience in `resume-recall.md`, not a different accomplishment wearing JD terminology;
- JD-critical keywords appear naturally in the most relevant experience/project bullets and/or Technical Skills;
- the document uses conventional ATS-friendly headings and plain, parsable terminology without keyword stuffing, graphics, or layout changes that obscure content;
- facts that must not change (name, contact information, employers, titles, dates, degree) remain accurate;
- the LaTeX syntax and existing command structure remain valid;
- `resume.tex` was compiled locally with TeX Live and the generated `resume.pdf` was checked to contain exactly one page; if it overflowed, lower-value or redundant content was cut and the file was rebuilt before finishing;
- only `resume.tex` has been modified.

## Final response

After editing, provide a brief report containing:

1. A one-sentence statement that only `resume.tex` was intentionally edited (apart from TeX Live build output), that `resume.pdf` was generated and verified as one page, and that it is a fictional, private target-state resume—not suitable for applications.
2. The JD themes and keywords that were emphasized.
3. A concise list of the most meaningful fictional changes.
4. A brief ATS rationale explaining how the most important JD requirements were surfaced for parsing and relevance.
5. A **bullet-molding map** for every materially changed bullet, using this exact compact format: `Source recall bullet → tailored resume bullet → preserved base (problem/action/result) → added JD-aligned layer → what I would need to do to make the addition true.` Quote or identify the source bullet clearly enough that I can find it in `resume-recall.md`.
6. A “development roadmap” mapping each invented or materially expanded claim—and every JD requirement omitted because it lacked a credible base—to the concrete skills, projects, responsibilities, and measurable outcomes I would need to build before I could truthfully put that claim on a real resume.

Do not ask clarifying questions. Make the best target-state modeling decision from the provided job description and repository files.
