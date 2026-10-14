# /code-review

Operator documentation for the `/code-review` skill in the han plugin. This document helps you decide _when_ and _how_
to use the skill. For what the skill does internally, read the skill definition at
[`han-coding/skills/code-review/SKILL.md`](../../skills/code-review/SKILL.md).

> See also: [Plugin README](../../README.md) · [Repo root](../../../README.md) · [All skills](../../../docs/skills/README.md) ·
> [All agents](../../../docs/agents/README.md) · [YAGNI](../../../docs/yagni.md)

## TL;DR

- **What it does.** Reviews a branch, uncommitted work, or named files for one named concern, such as security or
  testing, with the specialist agents that review type names, or runs the default review across all of them.
- **When to use it.** You want a principled review of what changed before merge: correctness, testing, security,
  documentation compliance, and project-pattern deference. Name a review type as the first word
  (`/code-review testing`) to focus on one concern, and run several typed reviews of one branch side by side.
- **What you get back.** A review report file with findings classified as CRIT / WARN / SUGG, each with
  `file_path:line_number` references and suggested fixes. A typed run's report name ends in `--{type}`, and the
  specialists' working files sit in a run folder beside it.

## Key concepts

- **Three severity levels.** CRIT (must fix before merge: security, data corruption, breaking API), WARN (should fix:
  bugs, missing error handling, missing tests), SUGG (consider: style, refactoring, docs).
- **One default severity rule, which a review type may override.** Critical and Warning findings escalate, and a
  Suggestion is raised only when this change directly introduces it. When a finding sits between two severities, the
  lower one wins. Manual findings (Steps 4 to 6) and agent findings (Step 7) follow the same rule, which lives in the
  Step 3.3 calibration directive. A review type's `## Severity` section can replace the rule for that type's runs; all
  six shipped types keep the default, and the default run always uses it.
- **Review types.** A review type is a small data file naming the specialists a review runs, the checklist categories
  its manual pass covers, its severity rule, any extra instructions for every agent, and any report section it adds.
  Han ships six, listed under [Review types](#review-types) below, and a team can declare more in `.han/config.md`. A
  typed run's report never names or recommends another review type.
- **Review modes on two independent axes.** The git axis: Mode A uses the full git branch diff, Mode B reviews
  uncommitted work when no branch diff exists, and Mode C reviews specified files when git is absent. **In Mode B and
  Mode C the YAGNI checklist is skipped unless explicitly requested**, because no diff exists to distinguish introduced
  code from pre-existing code. The agent axis: **manual-only mode** is the state a run enters when the dispatch
  mechanism fails (the `Agent` tool is unavailable or a dispatch call is denied), so no specialist reads the change. It
  is independent of the git axis; a Mode A run with a full branch diff can be manual-only. An agent that returns with
  nothing to say does not trigger it, because the security agent stays silent by design when its evidence bar is not
  met. In manual-only mode the manual review also sweeps, by hand, the checklist categories that substitute for each
  absent agent, and the report says which coverage was absent and which of it the sweep could not recover.
- **Packaging findings name a gap, not a defect.** When a Mode A diff changes what gets packaged (shading or relocation
  rules, vendoring, include or exclude patterns, dependency scope, bundling config), the checklist raises a Warning
  saying the review did not open the built artifact and what to check by hand before merging. Its summary row opens
  with `Not checked —` so you can tell it from a proven defect while triaging. It does not inspect the artifact.
- **A location is confirmed, not carried forward.** When the review reads a file over 1000 lines by its changed regions,
  a finding's location can be attributed upward to the wrong declaration. Such a finding names its enclosing unit and
  the lines the review read in isolation to confirm it. A finding from a file read whole carries no such note.
- **Agent dispatch by review type.** A typed run dispatches every agent on its type's `Always` line, plus each agent on
  its `When relevant` line whose signal appears in the changed files. The default run keeps the roster rule code review
  has always had: `junior-developer` (clarity and standards) and `adversarial-security-analyst` (exploit-path security)
  always run, and the rest of the default set's agents (`test-engineer`, `edge-case-explorer`, `structural-analyst`,
  `behavioral-analyst`, `concurrency-analyst`, `data-engineer`, `devops-engineer`, `on-call-engineer`,
  `observability-engineer`) join only when what the changed files touch matches their signal. Each agent's signal,
  file-list slice, output file, and finding-ID series live in one roster table in the skill's `agent-dispatch.md`.
- **Calibration directive.** Every dispatched agent receives a calibration directive that requires findings to be either
  introduced or worsened by the change, or critical irrespective of who introduced it. Theoretical concerns,
  pre-existing best-practice gaps, and benign-outcome scaling worries are excluded. The directive opens by naming the
  review type and carries the run's severity rule.
- **Agents hand back findings however they can.** Each prompt asks the agent to write its output to a named file in the
  run folder and say so, or to return its complete output as its final message. The skill reads the file the agent
  names, or saves the returned message into the run folder itself, and reads only what this run recorded. A rerun never
  reads an earlier run's files, and findings from agents that cannot write files still reach the report.
- **Per-agent dispatcher tailoring.** When `/code-review` dispatches `structural-analyst` and `behavioral-analyst`, it
  appends a default-SUGG directive so those agents start at the lowest severity and escalate only when the change
  introduces or worsens the issue. When it dispatches `junior-developer` and `edge-case-explorer`, it appends a
  file-list scoping directive so findings concern code on the scoped file list (with a narrower wording for
  `edge-case-explorer` that preserves its caller-read protocol while keeping the failure-mode target on the file list).
  These directives are `/code-review`'s tailoring; the agents' default behavior in other skills is unchanged.
- **Reachability phrase-match gate (Step 7.2).** When an agent's own rationale contains phrases like _theoretical_,
  _hypothetical_, _defense-in-depth_, _effectively impossible_, _in case the upstream_, _could happen_, _should never
  happen_, or _edge case that does not occur_, the finding is demoted by one severity before the rubric is applied.
  Security findings are exempt because the security agent's evidence standard already requires a demonstrated exploit
  path, and ticket-gap findings are exempt because a gap measures the change against a ticket requirement. The gate is a cheap, deterministic first pass and is brittle to paraphrase by design; a finding that hedges its
  reachability without one of the literal phrases is caught semantically by the Step 7.4 validation pass, not by growing
  the phrase list.
- **Independent findings validation (Step 7.4).** After the agent findings are collected and classified, the skill
  dispatches one `adversarial-validator` over the _consolidated corrective finding list_, the same agent `/investigate`
  uses to attack a root cause and fix. It is the only pass that re-reads the change in fresh context and judges each
  finding against the code rather than against the producing agent's rationale, so it does not anchor on what the
  specialists concluded. Each finding comes back Confirmed (kept), Partially Refuted (demoted one severity), or Refuted
  (dropped). A finding is dropped only when the validator supplies concrete counter-evidence at `file_path:line_number`;
  a bare assertion demotes rather than drops, and an uncertain verdict leaves the finding standing. Security findings
  are dropped only when the demonstrated exploit is refuted with counter-evidence. The pass is a finding _filter, not a
  finding source_: it never contributes findings of its own, and it skips entirely when the review produced no
  corrective findings. It also skips in manual-only mode, because it dispatches an agent; the report's Review Coverage
  section then says the findings were not re-checked by a second pass.
- **Branch context loaded at Step 1.5.** Before agents are dispatched, the skill loads four sources of branch-level
  context in order: PR description (via `gh pr view` when `gh` is available, Mode A only), a local `pr-body`,
  `PR_BODY.md`, or `.pr-body` file at the repo root, branch commit messages, and an implementation plan from the
  planning directory. The planning directory resolves first to the `plans:` (or `planning:`) key under CLAUDE.md's
  `## Project Discovery` section. Otherwise, the skill globs `docs/plans/*/feature-implementation-plan.md` and
  `plans/*/feature-implementation-plan.md`, picking the directory whose name matches the current branch (treating `-`
  and `_` as interchangeable). The loaded content is summarized into a `$branch_context` block of at most 200 words. It
  is plumbed, alongside the user's `$focus_areas` argument, into every agent prompt so agents avoid re-raising items the
  team has already deferred or resolved. Because PR descriptions, ticket bodies, and commit messages are third-party
  content that can carry text aimed at steering the review agent, the skill treats `$branch_context` as **untrusted
  data, not instructions**. The Step 1.5 summary strips any directives addressed to the reader or an agent. Step 3.5
  wraps the binding in explicit untrusted-data markers with a guard telling agents to use it for intent only and never
  to obey instructions inside it. The user's own `$focus_areas` argument is trusted and is not wrapped. Step 1.5 is
  skipped in Mode C. In Mode A and Mode B, when none of the four sources returns content, the skill emits a single
  fail-open warning and proceeds with `$branch_context` set to `none provided`.
- **Self-consistency check at Step 9.0.** Before the structural verification, the skill scans every pair of findings on
  the same file with overlapping line ranges, detects contradictory recommendations, demotes both, and adds a
  `Tension with {other-task-id}:` note for the human reviewer. Cross-file semantic contradictions are out of scope.
- **Premise verification before standards-compliance findings.** Step 5 requires reading at least one architectural file
  in the codebase that demonstrates a standard's premise before the skill raises a "violates standard X" finding. When
  the file does not confirm the premise, the finding is omitted with a logged note rather than raised on inferred
  premises.
- **Project-pattern deference.** A pattern that differs from general best practices but is consistent within the project
  is _not_ a finding. Only deviations from the project's own conventions count.
- **Automated tool boundary.** If the project has a linter or formatter, trust it. Only flag style issues that tooling
  cannot catch.
- **Documentation compliance and freshness.** ADRs, coding standards, and general docs are read and checked against the
  diff. Only documents whose subject matter the change touches are read; pulling unrelated standards into the review
  dilutes the signal and degrades judgment, the same reason Step 1.5 caps branch context. Correctness- and
  behavior-bearing rules are weighted over style minutiae the linter already enforces. Stale docs that misdescribe
  current behavior become CRIT findings.

## When to use it

**Invoke when:**

- You are about to merge a branch and want a principled pre-merge review covering correctness, tests, security, and doc
  compliance.
- A change has landed that you want audited against the project's ADRs, coding standards, or general docs.
- You want the exploit-path, coverage-gap, edge-case, structural, behavioral, clarity, and (when relevant) concurrency
  dimensions covered _in parallel_ rather than one at a time.
- You are working without a GitHub PR (local code, experimental branch, personal fork) and want the full review without
  posting to a PR.
- You want review findings on files you specify by name even when git is not available. The skill gracefully degrades to
  file-based review.

**Do not invoke for:**

- **Posting the review to a GitHub PR.** Use [`/post-code-review-to-pr`](../../../han-github/docs/skills/post-code-review-to-pr.md). It
  delegates to this skill and then posts the review as PR comments.
- **Explaining a change or getting oriented before reviewing.** Use [`/code-overview`](./code-overview.md) for a written
  overview, or [`/code-walkthrough`](./code-walkthrough.md) to be paced through the change one step at a time. Run
  either to understand what a change does, then run this skill to judge whether it is any good.
- **Architectural analysis.** Use [`/architectural-analysis`](./architectural-analysis.md) for coupling,
  data flow, concurrency, and SOLID assessment across a module.
- **Bug investigation.** Use [`/investigate`](./investigate.md) to find a root cause with evidence and adversarial
  validation.
- **Mapping bounded contexts or domain boundaries.** Use [`/ddd-analysis`](../../../han-ddd/docs/skills/ddd-analysis.md)
  to discover where the domain's boundaries sit and where the code's boundaries diverge from them. This skill judges the
  code in front of it; that one maps the domain the code is trying to express.
- **Test planning in isolation.** Use [`/automated-test-planning`](./automated-test-planning.md) when you want a prioritized test plan
  without a full correctness review, or [`/manual-test-planning`](./manual-test-planning.md) for a plain-language plan
  a person runs by hand.
- **Plan review.** Use [`/iterative-plan-review`](../../../han-planning/docs/skills/iterative-plan-review.md) for reviewing a work plan,
  not code.
- **Feedback on Han's own skills.** Use [`/han-feedback`](../../../han-feedback/docs/skills/han-feedback.md) to capture post-session
  feedback on the Han skills you ran.

## How to invoke it

Run `/code-review` in Claude Code. Pass an optional review type and an optional focus hint.

Give it:

1. **A review type, optional.** Name it as the first word: `security`, `testing`, `architecture`, `observability`,
   `maintainability`, `work-ticket <ticket>`, or a type your `.han/config.md` declares. Without one, the skill runs the
   default review. The first word `default` selects the default review explicitly. See [Review types](#review-types).
2. **A focus-area or context hint, optional.** _"Focus on the security implications of the new auth endpoints,"_ or
   _"review with extra attention to the database migration."_ Focus hints bias the manual review toward the named area;
   the parallel agents still run on their domain-scoped slice of files.
3. **A branch (implied).** If you are on a feature branch, the skill reviews changed files against the default branch
   (Mode A). If you are on the default branch with uncommitted work, Mode B kicks in. If there is no git, Mode C reviews
   what you point it at.
4. **Specific files or globs, optional.** In Mode C (or when you want to scope a git-mode review), pass file paths or
   glob patterns after any type. A path named like a type is written `./testing` so it reads as a path.

Example prompts:

- `/code-review`. The default review of the current branch's changes.
- `/code-review testing focus on retries`. A `testing` review, with "focus on retries" as the focus hint.
- `/code-review security the new auth endpoints`. A `security` review focused on the auth endpoints.
- `/code-review work-ticket docs/tickets/my-ticket.md`. Check whether the branch resolves that ticket. The ticket can
  also be a URL or an ID, such as `work-ticket PROJ-42`.
- `/code-review auth endpoints`. "auth" is not a review type, so this is the default review with "auth endpoints" as
  the focus hint.
- `/code-review src/billing/`. Scope the review to the billing directory.

**Size words are gone.** `small`, `medium`, `large`, and `dynamic` no longer do anything. A size word that is the whole
argument, or comes right before a type, is dropped with the note
`Size options were removed from code review; "{word}" was ignored.` Anywhere else it stays part of your focus text, so
`/code-review large file uploads` reviews with the focus "large file uploads".

**The run says what it understood before it starts.** At the end of Step 1, before any check runs, it prints the review
type and its purpose plus your focus text. A default review also lists the available types and, when your first word was
read as focus text, says so, so a mistyped type is caught before anything runs.

## What you get back

A structured review written to a file, named `code-review-{slug}.md` for a default review or
`code-review-{slug}--{type}.md` for a typed one, where the slug is the branch, ticket, or scope it covers, lowercased
with other characters turned into hyphens. It lands under the `output-directory` in your `.han/config.md` when you have
set one, and in the project's docs folder when you have not. Beside it sits a run folder of the same name
(`code-review-{slug}/` or `code-review-{slug}--{type}/`) holding each specialist's working file, and for `work-ticket`
a `ticket.md` copy of the ticket that was checked. The run prints that folder's path before dispatch and names it again
when it finishes. Because every run has its own names, two typed reviews of one branch never overwrite each other. A report already sitting at that name is replaced, and the run tells you which one it
replaced. When the resolved destination cannot be written, the run falls back and names the destination it could not
use, rather than throwing away a finished review.

Each finding's prose appears exactly once (its finding block, or its full security
block; the summary-table row is an index, not a copy), and sections render only when they have content: a review of a
small change produces a small document. The Review Summary table and the Review Recommendation are always present; every
other section appears only when it has at least one item, and when several are present they keep a fixed order
(the type section, Review Coverage, Critical, Warnings, Suggestions, YAGNI, Security Vulnerabilities, Remediation,
What's Good). The document can contain:

- **A review type line** under the title: `**Review type:** {token}: {purpose}`, or `**Review type:** default`. It is
  always present, so two reports of one branch can be told apart.
- **A type section**, present only when the run's type adds one. `work-ticket` adds **Ticket Resolution**: the ticket,
  a verdict, and one row per ticket requirement with its status (Met, Partial, or Not met) and the evidence. See
  [`work-ticket`](#work-ticket) for how the verdict is reached. The section sits between two marker comments so
  `/post-code-review-to-pr` can carry it whole.

- **A Review Coverage section**, present only when the run was manual-only. It opens by saying agent dispatch was
  unavailable so no specialist read the change, then carries one row per absent coverage area in plain English: what
  was swept by hand and under which checklist categories, and what was not swept with what to do instead. Its absence
  means every planned coverage ran.
- **A Review Summary table** indexing every corrective finding and every security finding across categories (automated
  checks, correctness, testing, security, ADR/standard/docs compliance, documentation freshness), ordered by severity. A
  corrective finding's tier is carried by its task-ID prefix; a security finding shows its tier inline in the row (for
  example, `SEC-001 (Critical)`) so the table stands alone as the complete severity index. Each row also carries the
  finding's fix route, and a finding the review established may never fire says so in its row. Both cues are there so
  you can triage a long list before opening any single finding. Row order, severities, and finding identifiers are
  unchanged; the cues sit inside existing rows.
- **Critical findings** (🔴). Each with task ID (`CRIT-001`, `CRIT-002`, …), `file_path:line_number`, the issue, and the
  recommended fix. Each also opens with a plain-language explanation written for someone who will not open the file:
  what they could observe going wrong, what has to be true for it to happen, and how likely that is, said outright when
  the answer is that the finding may never fire. All three are answered, and an answer that is not in doubt takes a
  clause rather than a sentence. The guidance for the person who will open the file follows it, unchanged. Each finding
  then names how it gets fixed: test-first (`/tdd`) when a behavior is missing, restructure (`/refactor`) when the
  behavior is right and the shape is wrong, or by hand when the edit is small. The route is named, never started. The
  `[Category]` label is kept on a block only when it names content a standalone reader needs (an ADR violation naming
  the record, a standards violation naming the standard, or a security finding) and dropped for generic categories the
  table already carries.
- **Warnings** (🟡). Same structure with task ID `WARN-NNN`.
- **Suggestions** (🔵). Same structure with task ID `SUGG-NNN`.
- **Agent findings.** Coverage gaps (`T#`), edge cases (`EC#`), security findings (`SEC-NNN`), structural findings
  (`S#`), behavioral findings (`B#`), clarity findings (`JD#`), concurrency findings (`C#` when the concurrency analyst
  was dispatched), data findings (`D#` when the data engineer was dispatched), devops findings (`DV#` when the devops
  engineer was dispatched), on-call resilience findings (`OCE#` when the on-call engineer was dispatched),
  observability findings (`OBS#`), ticket gaps (`GAP-###` on a `work-ticket` run), and findings from a team's own
  agents (`X-{agent name}`). Each is
  classified into the main severity tiers using the classification rubric. Security findings are the exception: they are
  not folded into a severity section (see below).
- **YAGNI findings.** Listed in their own `### 🟡 YAGNI` section with task IDs `YAGNI-NNN`. The section opens with the
  verbatim statement _"These findings will not be corrected unless explicitly requested. They are documented so the team
  can decide consciously whether to keep, simplify, or defer the items."_ Each finding is one line naming the failing
  evidence type, the matched anti-pattern, and a single reopen-trigger clause. YAGNI findings are advisory; they are not
  counted under CRIT / WARN / SUGG, do not appear in the summary table, and do not block a clean review. They carry no
  plain-language explanation either, because that reopen trigger already answers what the explanation would say.
- **Security vulnerabilities** (🔐). One full `SEC-NNN` block per proven vulnerability (OWASP category, location,
  evidence, `EXPLOIT:` path, and severity), followed by a single short **Remediation** note that references the
  `SEC-NNN` IDs and states the actionable fix in one or two sentences. Security findings are not cross-referenced into
  the Critical section; instead the Review Recommendation reflects their severity (a Critical-severity security finding
  yields a do-not-merge recommendation). The whole section is omitted when no proven vulnerabilities exist.
- **What's Good** (✅). Rendered only when there is a specific, substantive positive worth recording; omitted entirely
  otherwise rather than filled with generic praise.
- **Deferred tests note.** Test cases the `test-engineer` considered but excluded as brittle, listed for transparency
  (not counted toward the finding cap).
- **ADR / coding-standard / documentation compliance findings.** Violations of project-specific docs, tagged with the
  source (for example, `[ADR: 0042]`, `[Standard: error-handling]`, `[Docs Update: payments.md]`).

Finding caps are 30 items each for the manual review pass and the agent pass; security findings are not capped. If a cap
is exceeded, the skill says so and recommends another review after fixes land.

## How to get the most out of it

- **Run `/project-discovery` first.** The skill reads CLAUDE.md and `project-discovery.md` to find the ADR,
  coding-standards, and documentation directories. Without them, the compliance and freshness steps degrade to
  best-effort discovery.
- **Keep docs, ADRs, and standards up to date.** Every reference the skill finds sharpens the compliance check. Stale
  docs that contradict current behavior become CRIT findings, which is the signal to update the doc, not to bypass the
  skill.
- **Use focus hints for high-stakes branches.** When a branch touches a load-bearing surface (auth, billing, data
  migrations), name it in the prompt. The skill biases manual attention toward the area while the parallel agents still
  cover the full scope.
- **Pair with `/investigate` when findings reveal a bug.** If the review surfaces a CRIT finding whose root cause needs
  deeper analysis, dispatch `/investigate` next. It produces a fix plan with adversarial validation.
- **Pair with `/architectural-analysis` when findings reveal coupling or structural issues.** The review runs per-file;
  the architectural analysis runs per-module. Use both when the branch touches boundaries.
- **Re-run after fixes.** The skill is cheap to re-dispatch. Fix the findings, run again, confirm the count drops.
- **Use `/post-code-review-to-pr` if you want it posted to the PR.** `/post-code-review-to-pr` invokes this skill
  end-to-end, then posts the review to GitHub. If you already ran this one locally, you can run
  `/post-code-review-to-pr` next to publish.

## Review types

Six review types ship with the skill. Each is a file under `han-coding/skills/code-review/references/types/`.

| Type              | Always runs                                | Joins when its signal fires            | Manual pass covers                                                                                   |
| ----------------- | ------------------------------------------ | -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `security`        | `adversarial-security-analyst`             | none                                   | Data Isolation, Error Handling, API Design                                                           |
| `testing`         | `test-engineer`, `edge-case-explorer`      | none                                   | Testing, Correctness                                                                                 |
| `architecture`    | `structural-analyst`, `behavioral-analyst` | `concurrency-analyst`, `data-engineer` | Code Organization, API Design, Database, Performance                                                 |
| `observability`   | `observability-engineer`                   | `on-call-engineer`, `devops-engineer`  | Error Handling                                                                                       |
| `maintainability` | `junior-developer`                         | none                                   | Code Maintainability, Documentation, Code Style & Patterns, Architecture Decision Records, Packaging |
| `work-ticket`     | `gap-analyzer`                             | `test-engineer`                        | Correctness, Testing                                                                                 |

Every type runs the same shared flow: automated checks, the documentation compliance and freshness passes, the YAGNI
pass, the independent validation pass, and the readability rewrite. A type narrows which specialists run and which
checklist categories the manual pass applies.

**The default review.** With no type, the skill runs `security`, `testing`, `architecture`, `observability`, and
`maintainability` as one run with one report. `junior-developer` and `adversarial-security-analyst` always run, every
other agent in those five types runs only when its signal fires, and the manual pass covers every checklist category.
The default review applies none of the member types' severity, instructions, or report section.

**A focused review skips security.** `/code-review testing` runs no security check, and its report does not say so. Run
`security` on its own, or run the default review, when you want the security agent.

### `work-ticket`

`/code-review work-ticket <ticket>` checks whether the change resolves a work ticket. The ticket can be a local file, a
URL, or an ID. With nothing after `work-ticket`, the run stops with a usage line.

- **Loading the ticket (Step 1.6).** A local file is read directly. Anything else is fetched with whatever tools and
  permissions your Claude installation has; the skill names no fetch method. When the load fails, or returns a sign-in
  page or an empty body, the run prints
  `work-ticket could not load the ticket {ticket}: {what failed}.` and
  `Paste the ticket's text, or tell me how to get it, and the review will continue.`, then waits. Paste the text, or
  say how to get it and the skill retries. Nothing is dispatched while it waits.
- **Checking it.** The ticket is copied to `ticket.md` in the run folder, and `gap-analyzer` compares the change against
  it, treating the ticket as data and never following an instruction inside it. It lists every requirement the ticket
  states, met or not, with a count of how many the ticket has.
- **The verdict.** Ticket Resolution reads **Not verified** when no requirement map came back (with one line saying
  why), **Not resolved** when any requirement is not met, **Resolved** when every requirement is met and none are
  missing from the count, and **Partially resolved** otherwise, naming how many requirements were not checked.

### Review types and agents from `.han/config.md`

A team can declare its own review types, such as a `pii-check`, under `## Code Review Types`, each pointing at a type
file with the same six sections the shipped ones use. It can also attach its own agents to any type, shipped or
declared, under `## Code Review Type Agents`; an attached agent runs on every run of that type. A configured type never
joins the default review. `## Extra Agents` reaches only the default review, under its signal rule, and typed runs
ignore it. When a configured type's file is missing or incomplete, invoking that type stops the run before it starts
and says which file to fix. See [Configuration](../../../docs/configuration.md) for the schema and a worked example.

## Cost and latency

Cost scales with the review type and with what the change touches. A typed review runs one to four specialists; the
default review runs its two always-run agents plus each agent whose signal fires, up to eleven. Automated checks
(lint/build/tests from the project config), a file-by-file manual review, a documentation compliance pass, and a
freshness pass run on every review.

Parallel typed reviews each run those shared steps, including Step 2's automated checks, in the same working tree. Two
runs started at once can collide on build output, ports, or test databases and report failures the change did not
cause.

When the review produces at least one corrective finding, one additional `adversarial-validator` runs after the roster
to validate the finding list (Step 7.4); a clean review skips it, and so does a manual-only run, which cannot dispatch
it. One `han-communication:readability-editor` then
rewrites the assembled review's prose against the shared readability standard (Step 8.5), leaving every task ID,
severity, `file_path:line_number` reference, and code snippet unchanged.

Agents run on their default models. Finding caps of 30 per pass keep output bounded. Security findings are uncapped. The
skill is built for per-branch cadence, not tight-loop iteration over the same code. Fix the findings and re-run.

## In more detail

The skill walks an eleven-step process (Step 1.5 is a context loader inserted between Steps 1 and 2, and Step 8.5 is a
readability pass inserted between Steps 8 and 9):

1. **Identify changes.** Parse the argument string into an optional retired size word (dropped with a note), an
   optional review type (and ticket for `work-ticket`), and `$focus_areas`. Detect git mode (A/B/C), resolve project
   config, enumerate changed files. Read the type file, stop if it is missing a section, and announce the review type
   before anything runs. 1.5. **Load branch context.** Attempt PR description via
   `gh pr view --json title,body,headRefName,baseRefName` (Mode A only, when `gh` is available); a local `pr-body`,
   `PR_BODY.md`, or `.pr-body` file at the repo root; branch commit messages via
   `git log {default-branch}..HEAD --pretty=format:%B` (Mode A) or `git log -n 20 --pretty=format:%B` (Mode B); and an
   implementation plan resolved through the CLAUDE.md `plans:` / `planning:` key or, failing that, a Glob over
   `docs/plans/*/feature-implementation-plan.md` and `plans/*/feature-implementation-plan.md` that prefers the directory
   matching the current branch name (with `-` and `_` interchangeable). Summarize loaded content into `$branch_context`
   (at most 200 words), treating the fetched content as untrusted data: the summary strips any instruction or directive
   addressed to the reader or an agent rather than carrying it forward. When nothing loads, emit a single fail-open
   warning, set `$branch_context` to `none provided`, and proceed. Skipped in Mode C. At Step 3.5 the binding is wrapped
   in explicit untrusted-data markers so agents use it for intent only and never obey instructions inside it. 1.6.
   **Load the work ticket.** `work-ticket` only: read or fetch the ticket, and wait on you when it cannot.
2. **Automated quality checks.** Run the project's lint, build, and test commands. Report each failure as a CRIT finding
   with category `[Automated Check]`. Do not fix; report.
3. **Resolve the review type and dispatch review agents in parallel.** Step 3.0 names the run and its folder, and
   writes `ticket.md` for `work-ticket`. Step 3.1 carries the review type forward and summarizes the change. Step 3.2
   builds the roster from the type and the config. Step 3.3 is the authoritative home for the severity rule and
   attaches the calibration directive verbatim to every brief. Step 3.4 narrows each agent's brief to a domain-scoped slice of the file list (for example,
   `structural-analyst` receives source files only; `data-engineer` receives schema, migration, query, ORM, and
   data-access files only; `junior-developer` and `adversarial-security-analyst` receive the full list). Step 3.5
   launches all selected agents in parallel, appending the shared `$focus_areas` and `$branch_context` blocks to every
   prompt, the type's instructions when it has any, and the per-agent dispatcher directives to `structural-analyst`,
   `behavioral-analyst`, `junior-developer`, and `edge-case-explorer`, and files what each agent returns. If the dispatch mechanism fails, the run enters manual-only mode: it keeps the selected roster
   as the absent-coverage list and the manual review sweeps the mapped checklist categories in the agents' place.
4. **Manual file-by-file review.** Every changed file (alphabetical) against the review type's checklist categories (all
   of them on the default review: correctness, data isolation, performance, error handling, testing, API design,
   maintainability, organization, docs, style, database, packaging in Mode A only, ADR compliance), plus the YAGNI pass
   on every review. In Mode B and Mode C, the YAGNI checklist is skipped unless the user
   requests it in `$focus_areas`. A finding located through a region read of a file over 1000 lines has its enclosing
   unit confirmed by reading that unit in isolation before the finding is written.
5. **Documentation compliance analysis.** Read only the ADRs, coding standards, and docs whose subject matter the change
   touches, not the whole directory, and weight correctness-bearing rules over style minutiae the linter already covers.
   Verify each standard's premise applies by reading at least one architectural file in this codebase before raising a
   "violates standard X" finding; omit findings whose premise is not verified.
6. **Documentation freshness review.** Check whether docs describing the changed code are now stale.
7. **Collect, classify, and validate findings in four sub-steps.** 7.1 reads the source recorded for each dispatched agent in this run. 7.2
   applies the reachability phrase-match demotion gate (CRIT → WARN → SUGG → omitted) when a finding's rationale
   contains a documented reachability phrase; security and ticket-gap findings are exempt. 7.3 classifies the surviving
   findings using the rubric in `agent-finding-classification.md`, governed by the Step 3.3 calibration directive. Junior-developer
   findings that overlap with a specialist's finding reference the specialist instead of duplicating. 7.4 dispatches one
   `adversarial-validator` over the consolidated corrective finding list to confirm, demote, or (with concrete
   counter-evidence) drop each finding; it runs whenever any corrective finding exists, and skips when the review is
   clean or the run is manual-only. Its brief also challenges any finding whose location came from a region read and
   names an enclosing unit never read in isolation.
8. **Generate review output.** Assemble the final review using the review template, rendering each section only when it
   has content and keeping the fixed section order.
   8.5. **Rewrite the finding prose for readability.** Dispatch `readability-editor` over the assembled review so its prose
   meets the shared readability standard, with every task ID, severity, `file_path:line_number` reference, and code excerpt
   preserved.
   8.6. **Write the report file.** Write it under the name Step 3.0 gave the run, beside the run folder. A report already at that
   name is replaced and recorded; an unwritable destination falls back and is recorded.
9. **Verify.** Step 9.0 runs the self-consistency check (extract
   `{task-id, file-path, line-range, recommended-action-summary}` tuples, then compare overlapping pairs and demote
   contradictory recommendations with a `Tension with {other-task-id}:` note). Step 9.1 then verifies task IDs are
   sequential, `file_path:line_number` references are valid, exploit fields are populated for security findings, the
   summary table indexes every corrective and security finding (with security tiers shown inline) and matches the
   sections present, no section is rendered empty, security findings carry no Critical cross-reference while the
   recommendation still reflects their severity, and the YAGNI section's verbatim opening is preserved. The same step
   confirms the newer content arrived: every finding you are expected to act on carries its plain-language explanation,
   every corrective one names a fix route, and a finding that may never fire says so both in its own explanation and in
   its summary row. It checks that a manual-only run's report carries the Review Coverage section with one row per
   absent area, and that a full-coverage run's report does not, and that every region-read finding names its confirmed
   enclosing unit. Anything missing is fixed before the review reaches you, not reported to you as a caveat.
10. **Present.** A short message that leads with the recommendation and the counts by severity, then the path, then the
    run's own facts. The review is never pasted into the conversation.

## What the run says when it finishes

A short message, in a fixed order, so the answer is the first thing you read:

1. The recommendation, in the report's own words.
2. The counts by severity, with any advisory count named separately rather than folded into the total.
3. The path to the report and to the run folder, plus any report it replaced and any destination it could not write to.
4. The run's own facts last: the validator reconciliation. When no specialist read the
   change, this part says so: agent dispatch was unavailable, how many coverage areas were swept by hand and how many
   were not, and that the report's Review Coverage section lists them.

A clean review says the code can be approved and gives you the path. A review whose only findings are advisory still
recommends approval, says the count needing action is zero, and names the advisory count beside it, so you are never
told "no findings" about a report whose body lists items.

## YAGNI

YAGNI in `/code-review` is **advisory-only** and runs as a two-pass procedure. **Pass 1, evidence test:** for every
speculative addition (defensive code, single-implementation interfaces, configuration knobs no caller sets,
instrumentation for non-flowing telemetry), check whether the diff contains evidence of need from one of the acceptable
evidence types in [`yagni-rule.md`](../../references/yagni-rule.md). When evidence is present, do not
flag. **Pass 2, anti-pattern check:** only items that fail Pass 1 are matched against the named anti-patterns; matches
become `YAGNI-###` findings whose body names the failing evidence type, the matched anti-pattern, and the simpler form
considered.

A YAGNI finding alone does not block a clean review; the posture is _make the cost of inclusion visible_, not _reject
the change_. Critical-path correctness, security, and data-integrity findings are unaffected by this advisory posture
and follow the standard severity rules. In Mode B and Mode C, the YAGNI checklist is skipped unless the user explicitly
requests it, since the diff signal that distinguishes introduced code from pre-existing code is absent.

See [YAGNI](../../../docs/yagni.md) for the two gates, the acceptable-evidence list, the named anti-patterns, and why review
skills make the cost of inclusion visible rather than enforce inclusion bans.

## Sources

The skill's protocols are grounded in established practice for pre-merge code review, parallel adversarial specialist
review, and documentation-compliance checking.

### Karl E. Wiegers: Peer Reviews in Software

Wiegers's _Peer Reviews in Software_ (2001) formalized inspection-style peer review as a distinct engineering
discipline: a structured, checklist-driven pass that finds defects earlier and cheaper than testing alone. The skill's
manual review step uses a structured checklist for exactly this reason.

URL: https://www.processimpact.com/books/PeerReviews.html

### Smartbear: State of Code Review

Smartbear's annual surveys of code-review practice document the measurable effect of structured reviews on defect
density and time-to-fix. The skill's severity scheme (CRIT/WARN/SUGG) and its finding cap reflect the consistently
measured finding that reviews of more than ~400 LoC per hour lose effectiveness. Caps prevent flood and prioritization
drift.

URL: https://smartbear.com/resources/ebooks/the-state-of-code-review/

### Gene Kim, Jez Humble, et al.: Accelerate

_Accelerate_ documents DORA research showing that high-performing teams rely on fast, automated, continuous review
practices as part of their delivery flow. The skill's parallel agent dispatch during the manual review reflects this.
Specialists and the reviewer work concurrently rather than in sequence.

URL: https://itrevolution.com/product/accelerate/

## Related documentation

- [Plugin README](../../README.md). The plugin's front door: its skills, agents, and how they fit together.
- [Repo root README](../../../README.md). The Han suite landing page. Start here if you arrived from outside the docs tree.
- [YAGNI](../../../docs/yagni.md). The evidence-based "You Aren't Gonna Need It" rule this skill applies before committing
  items. The two gates, the acceptable-evidence list, the named anti-patterns, and the deferral format.
- [Skills Index](../../../docs/skills/README.md). All skills, grouped by purpose.
- [`/post-code-review-to-pr`](../../../han-github/docs/skills/post-code-review-to-pr.md). Wraps this skill and posts the review to a
  GitHub PR.
- [`/code-overview`](./code-overview.md). The orientation counterpart: run it to understand a change before this skill
  judges its quality.
- [`/code-walkthrough`](./code-walkthrough.md). The paced counterpart to `/code-overview`: run it when you want to be
  taught the change step by step before this skill judges it.
- [`/investigate`](./investigate.md). Next step when a CRIT finding hides a bug whose root cause needs deeper analysis.
- [`/architectural-analysis`](./architectural-analysis.md). Run alongside when the change touches module
  boundaries.
- [Configuration](../../../docs/configuration.md). The `.han/config.md` guide, including `## Code Review Types` and
  `## Code Review Type Agents`.
- [`junior-developer`](../../../han-core/docs/agents/junior-developer.md),
  [`adversarial-security-analyst`](../../../han-core/docs/agents/adversarial-security-analyst.md). The two agents the
  default review always dispatches.
- [`gap-analyzer`](../../../han-core/docs/agents/gap-analyzer.md). The agent `work-ticket` always dispatches to compare
  the change against the ticket.
- [`test-engineer`](../../../han-core/docs/agents/test-engineer.md),
  [`edge-case-explorer`](../../../han-core/docs/agents/edge-case-explorer.md),
  [`structural-analyst`](../../../han-core/docs/agents/structural-analyst.md),
  [`behavioral-analyst`](../../../han-core/docs/agents/behavioral-analyst.md),
  [`concurrency-analyst`](../../../han-core/docs/agents/concurrency-analyst.md). Agents the review types name; each joins
  a default review when its signal appears in the file list.
- [`data-engineer`](../../../han-core/docs/agents/data-engineer.md),
  [`devops-engineer`](../../../han-core/docs/agents/devops-engineer.md). Conditional dispatches for changes touching
  schemas/migrations/queries (data) or infra/CI/observability (devops).
- [`on-call-engineer`](../../../han-core/docs/agents/on-call-engineer.md). Conditional dispatch when the change adds or modifies
  application source with runtime resilience surface (outbound calls, retry logic, queue/buffer handling, async/await
  code, error-handling on failure paths, idempotency, schema migrations co-deployed with dependent code, new production
  code paths). Hard boundary against `devops-engineer`: this agent reads application source only.
- [`observability-engineer`](../../../han-core/docs/agents/observability-engineer.md). Conditional dispatch when the
  change adds or alters telemetry design: metric instruments or labels, span attributes, log schemas, OpenTelemetry SDK
  setup, or telemetry pipeline configuration. SLOs, alert rules, and dashboards stay with `devops-engineer`;
  source-line debuggability stays with `on-call-engineer`.
- [`adversarial-validator`](../../../han-core/docs/agents/adversarial-validator.md). Dispatched once at Step 7.4 to re-attack
  the consolidated finding list against the code and confirm, demote, or drop each finding. Runs whenever the review
  produced at least one corrective finding.
- [`readability-editor`](../../../han-communication/docs/agents/readability-editor.md). Dispatched at Step 8.5 to rewrite the
  review's prose against the shared readability standard for the change's author and reviewers, leaving task IDs,
  severities, `file_path:line_number` references, and code snippets unchanged.
- [`SKILL.md` for /code-review](../../skills/code-review/SKILL.md). The internal process definition.
