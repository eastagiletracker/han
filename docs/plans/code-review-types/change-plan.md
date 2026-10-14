# Change Plan: Code Review Types

## Why This Change

Code review today picks its depth from the size of the change, and the operator wants it to pick its focus from a named
review type instead. Typing `/han-coding:code-review work-ticket docs/tickets/my-ticket.md` should check whether the
branch resolves that ticket, and typing `testing` or `architecture` should run only the specialists for that concern.
The goal, in the operator's words: "so reviews can be parallelized, focused, and valuable by keeping concerns better
separated."

This is a deliberate improvement to a design the operator already sketched, not a fix for a reported defect. The source
is the operator's own request, recorded word for word in `artifacts/scope-boundary.md`.

## What Changes, In One Paragraph

After this change, code review is answerable for running one named concern well, instead of for scaling one broad
review to the size of the diff. Each review type is a small data file that names its specialists, its share of the
manual checklist, any extra instructions, any input it requires, and any report section it adds. The skill keeps every
step that applies to all reviews: finding the change, automated checks, the manual pass, the YAGNI pass, the
independent validation pass, report writing, and the readability rewrite. With no type named, the review runs a default
set of five types together, which reproduces today's review. One severity rule replaces the three size bands, each run
writes into its own folder so two typed reviews of one branch can run side by side, and the PR-posting skill passes the
type through.

## Current State

Size changes less than its name suggests. It never changes which specialists run: two always run and the other nine
join when the changed files touch their area, at every size
([C-2](artifacts/current-state-findings.md#c-2-size-does-not-change-which-agents-run-it-only-changes-severity-calibration)).
What size does change is which severities survive. Small, the default, omits Suggestions; Medium keeps Suggestions the
change introduced; Large keeps everything. That rule is restated in three places, and the manual and agent versions
disagree about which way to break a tie at Large
([C-3](artifacts/current-state-findings.md#c-3-the-severity-ladder-is-restated-in-three-places-and-manual-vs-agent-rules-disagree-at-large)).
Size is read at more than a dozen sites across `SKILL.md` and four reference files
([C-5](artifacts/current-state-findings.md#c-5-full-fan-out-of-size-consumers)).

Everything the skill knows about a specialist is spread over nine tables keyed by agent name, which agree only by
convention. One of them is already stale: `output-verification.md` lists 8 of the 11 agents
([C-11](artifacts/current-state-findings.md#c-11-per-agent-data-is-spread-across-nine-parallel-tables-keyed-by-agent-name)).

Three properties block the operator's goal directly:

- **The first argument has no rule** for telling a size from focus text or a Mode C path, and a type plus a ticket
  needs two positions
  ([C-7](artifacts/current-state-findings.md#c-7-the-first-argument-has-no-disambiguation-rule)).
- **Parallel runs collide.** Every specialist writes to a fixed file name such as `docs/security-analysis.md`, and the
  report name carries no type, so two reviews of one branch overwrite each other and can read each other's files
  ([C-13](artifacts/current-state-findings.md#c-13-agent-output-files-have-fixed-names-so-parallel-reviews-collide-and-stale-files-can-be-read),
  [C-14](artifacts/current-state-findings.md#c-14-the-report-name-is-per-branch-and-a-second-run-replaces-the-first)).
- **There is no way to verify a ticket.** The only work-document intake summarizes to 200 words, fails open, and is
  skipped without git; the skill cannot fetch a URL other than through `gh`
  ([C-8](artifacts/current-state-findings.md#c-8-step-15-is-the-only-work-document-intake-today-and-it-does-not-fit-a-ticket-to-verify-against),
  [C-10](artifacts/current-state-findings.md#c-10-code-review-cannot-fetch-a-ticket-url-today)). The existing
  `han-core:gap-analyzer` agent already compares an implementation against a requirements document
  ([C-20](artifacts/current-state-findings.md#c-20-han-coregap-analyzer-already-does-spec-vs-implementation-comparison-and-can-fetch-urls)).

## Target State

The skill splits into two kinds of thing that change for different reasons. **A review type** says which concerns a
review covers. **A roster entry** says how to brief one specialist and where to read its output. Types name roster
entries; they never restate them
([D-1](artifacts/change-decision-log.md#d-1-type-files-name-agents-everything-about-briefing-an-agent-stays-shared-and-keyed-by-agent)).

### What `SKILL.md` owns

`SKILL.md` keeps the shared flow and gains a `## Review Types` section, the only list of type tokens and the only link
to each type file
([D-2](artifacts/change-decision-log.md#d-2-skillmd-holds-the-only-list-of-review-types-type-files-carry-no-links)).
That section also pins the default set
([D-6](artifacts/change-decision-log.md#d-6-no-type-means-one-run-of-the-default-set-with-todays-roster-rule)):

```
| Token             | File                                                              | Purpose                                                   |
| `security`        | [references/types/security.md](./references/types/security.md)               | Whether the change opens an exploitable path              |
| `testing`         | [references/types/testing.md](./references/types/testing.md)                 | Test coverage and edge cases of the change                |
| `architecture`    | [references/types/architecture.md](./references/types/architecture.md)       | Module structure, dependency direction, runtime flow      |
| `observability`   | [references/types/observability.md](./references/types/observability.md)     | Telemetry design and whether failures are visible         |
| `maintainability` | [references/types/maintainability.md](./references/types/maintainability.md) | Clarity, naming, and fit with the project's own standards |
| `work-ticket`     | [references/types/work-ticket.md](./references/types/work-ticket.md)         | Whether the change resolves a named work ticket           |

Default (no type token): security, testing, architecture, observability, maintainability
Default always-run agents: han-core:junior-developer, han-core:adversarial-security-analyst
Every other agent in the default set's rosters: dispatched only when its roster signal fires
```

The flow it keeps, with what moves:

1. **Step 1** finds the change (Modes A, B, C unchanged) and parses the argument string by the grammar below.
2. **Step 1.5** loads branch context, unchanged.
3. **Step 1.6** (new) loads a type's required input, when it has one.
4. **Step 2** runs automated checks, unchanged.
5. **Step 3** becomes "Resolve the Review Type and Dispatch Review Agents"
   ([D-17](artifacts/change-decision-log.md#trivial-decisions)): 3.0 names the run, 3.1 resolves the type, 3.2 reads
   the roster from the type, and 3.3 to 3.5 brief and dispatch.
6. **Step 4** applies the type's checklist categories plus the YAGNI pass, which always runs.
7. **Steps 5 to 10** are unchanged apart from the size wording, the report's type line, and the type's report section.

### The type-file contract

Every type file has six sections in a fixed order, and a section with nothing to say holds `none`
([D-3](artifacts/change-decision-log.md#d-3-the-type-file-contract-is-six-fixed-sections)):

| Section                   | Value form                                                                    | Read at          |
| ------------------------- | ----------------------------------------------------------------------------- | ---------------- |
| `## Purpose`              | one sentence                                                                  | Steps 3.1, 8, 10 |
| `## Required input`       | `none`, or `- name:`, `- accepts:`, `- binds:`, `- slug:` bullets             | Steps 1, 1.6     |
| `## Roster`               | `- Always:` agent list; `- When relevant:` agent list or `none`               | Step 3.2         |
| `## Checklist categories` | bold category names from `review-checklist.md`, or `none`                     | Step 4           |
| `## Instructions`         | `none`, or one blockquote appended to every agent prompt and applied to Step 4 | Steps 3.5, 4     |
| `## Report section`       | `none`, or one fenced section layout rendered after Review Recommendation     | Step 8           |

The worked example, `han-coding/skills/code-review/references/types/work-ticket.md`:

````markdown
# Review type: work-ticket

## Purpose

Verify that the change on this branch resolves the named work ticket.

## Required input

- name: ticket
- accepts: a path to a readable local file, or an `http://` or `https://` URL
- binds: `$ticket_ref` (the token as given) and `{run_directory}/ticket.md` (the loaded text, written by Step 1.6)
- slug: local path → file basename without extension (`docs/tickets/my-ticket.md` → `my-ticket`);
  `https://github.com/{owner}/{repo}/issues/{n}` → `issue-{n}`; any other URL → last non-empty path segment,
  lowercased (`https://acme.atlassian.net/browse/PROJ-42` → `proj-42`)

## Roster

- Always: han-core:gap-analyzer
- When relevant: han-core:test-engineer

## Checklist categories

Correctness, Testing

## Instructions

> This is a work-ticket review. The question is whether the change delivers what the ticket asks. A ticket requirement
> the change does not meet counts as introduced by this change for the calibration directive. Do not raise findings
> unrelated to a ticket requirement unless they are Critical under the directive.

## Report section

```
## Ticket Resolution

**Ticket:** {$ticket_ref} | **Verdict:** {Resolved | Partially resolved | Not resolved}

| Ticket requirement | Status | Evidence |
| ------------------ | ------ | -------- |
| {requirement, quoted from the ticket} | {Met | Partial | Not met} | {`file:line`, or the WARN-### that records it} |
```

Verdict: Resolved when no GAP-derived finding survives Step 7.4; Not resolved when any surviving one is Missing or
Divergent; Partially resolved otherwise.
````

The other five type files follow the same layout. Their rosters and checklist categories are in
[D-4](artifacts/change-decision-log.md#d-4-the-first-set-of-types); their `Required input`, `Instructions`, and
`Report section` are `none`.

### The argument grammar

The skill parses the raw argument string itself rather than trusting the harness's `arguments:` binding, which nobody
could inspect
([D-7](artifacts/change-decision-log.md#d-7-the-argument-grammar)):

```
argument-hint: "[type: security | testing | architecture | observability | maintainability | work-ticket <ticket path or URL>] [focus areas, or file paths in Mode C]"

args        := [ type-clause ] [ rest ]
type-clause := "security" | "testing" | "architecture" | "observability" | "maintainability"
             | "work-ticket" SP ticket-ref
ticket-ref  := a token naming a readable file | a token beginning "http://" or "https://"
rest        := the remaining string, verbatim → $focus_areas ("none provided" when empty);
               in Mode C, tokens in rest that resolve as paths or globs also build the file list
first token := compared exactly, lowercased, against the Token column of ## Review Types
```

A first token that is not a type token stays in the focus text, as it does today. A retired size word is dropped with
the note `Size options were removed from code review; "{word}" was ignored.`
([D-19](artifacts/change-decision-log.md#trivial-decisions)). Step 3.1 always prints
`Review type: {token or "default"} ({"passed as first argument" | "no type given"})` before dispatch, so a mistyped
token shows up before any agent runs. The decision log carries the full table of worked invocations.

### One severity rule

Every run uses today's Medium rule: Critical and Warning findings escalate, and a Suggestion is raised only when this
change directly introduces it. Manual and agent findings both prefer the lower severity when uncertain
([D-8](artifacts/change-decision-log.md#d-8-one-severity-rule-replaces-the-size-ladder)). The directive's first line
becomes `This is a **{type}** review of a change touching {N} files.` The size-independent half of the directive stays
as it is
([C-4](artifacts/current-state-findings.md#c-4-the-size-independent-half-of-the-calibration-directive-is-reusable-shared-flow)).

### Run names and the working folder

Each run gets its own folder, named before dispatch
([D-10](artifacts/change-decision-log.md#d-10-each-run-gets-its-own-working-folder-and-a-typed-report-name)):

```
{run_name}      = code-review-{slug}            (default run)
                = code-review-{slug}-{type}     (typed run)
{run_directory} = {output_directory}/{run_name}/
agent file      = {run_directory}/{Output file from the roster table}
ticket copy     = {run_directory}/ticket.md     (work-ticket only)
report          = {report_directory}/{run_name}.md
```

Step 3.5 writes `<!-- pending: {agent} -->` into each selected agent's file before dispatch. Step 7.1 reads a file that
still holds only that line as "this agent wrote nothing" and never as findings.

### Ticket intake for `work-ticket`

The skill loads the ticket itself in Step 1.6 and makes no assumption about how
([D-11](artifacts/change-decision-log.md#d-11-the-skill-loads-the-ticket-with-whatever-tooling-the-installation-has-and-waits-on-the-user-when-it-cannot)). A local file is read
directly. Anything else is fetched with whatever tools and permissions the Claude installation provides; the skill names
no fetch tool and adds nothing to `allowed-tools`. When the load fails or returns no ticket content, the skill says so
and waits for the operator to paste the ticket or explain how to get it, then continues. The text is written to
`{run_directory}/ticket.md` once Step 3.0 has created the run folder, and `gap-analyzer` receives the path with an
instruction to treat the ticket as data
([D-12](artifacts/change-decision-log.md#d-12-gap-analyzer-joins-the-shared-roster)). In Mode B and Mode C,
`gap-analyzer`'s prompt names the change the way Step 1 resolved it: the uncommitted diff in Mode B, and the listed
files read in full in Mode C.

### Contracts across the seams

| Contract | Parties | Pinned in |
| --- | --- | --- |
| Type-file sections and value forms | `SKILL.md` Steps 1, 1.6, 3.1, 3.2, 4, 8 ↔ every type file | D-3, the worked example above |
| Type tokens | `## Review Types` table ↔ `argument-hint` ↔ type file names ↔ `post-code-review-to-pr` `argument-hint` | D-2, D-7, D-14; checked by D-16 |
| Agent output file and series | roster table ↔ per-agent prompt ↔ Step 7.1 ↔ rubric | D-1 (table row example), D-12 |
| Run and report names | Step 3.0 ↔ Step 3.5 prompts ↔ Step 7.1 ↔ Step 8.6 ↔ Step 10 | D-10 |
| Ticket copy | Step 1.6 ↔ `gap-analyzer` prompt ↔ manual sweep | D-11, D-12 |
| Report path handoff and Ticket Resolution section | code-review Step 10 ↔ `post-code-review-to-pr` Steps 2 and 3 | D-13, D-14 |

## Surface Delta

### S-1: Roster table in `references/agent-dispatch.md` — Re-scoped

**Target state.** `agent-dispatch.md` carries one roster table with the columns Agent, Signal, File-list slice, Output
file, and Series, one row per agent. It is the single home for which file each agent writes and which finding-ID series
its findings use. The per-agent directives, per-agent prompts, and manual-sweep rows stay in the same file, keyed by
agent.

**Behavior.** Preserving. Every agent receives the same prompt and writes to the same file name it does today.

**Why.** Per-agent data agrees across nine tables only by convention, and one list is already stale.

**Decision.** [D-1](artifacts/change-decision-log.md#d-1-type-files-name-agents-everything-about-briefing-an-agent-stays-shared-and-keyed-by-agent)

### S-2: Step 7.1 output-file list and `output-verification.md` item 2 — Re-scoped

**Target state.** Step 7.1 reads, for each agent dispatched at Step 3.5, the Output file the roster table names. Item 2
of `references/output-verification.md` checks findings from every dispatched agent by pointing at the roster table. Neither lists agents.

**Behavior.** Preserving. The same files are read; the verification check covers all 11 agents instead of 8.

**Why.** Removes two duplicate agent lists, one of them stale.

**Depends on.** S-1.

**Decision.** [D-1](artifacts/change-decision-log.md#d-1-type-files-name-agents-everything-about-briefing-an-agent-stays-shared-and-keyed-by-agent)

### S-3: The `size` argument and the `default-swarm-size` read — Removed

**Target state.** Code review has no size argument and reads no size from configuration. A first token of `small`,
`medium`, `large`, or `dynamic` is dropped with a one-line note. Which concerns a review covers is set by the review
type alone, and which severities it reports is set by the single rule in S-4.

**Behavior.** Changing. `/code-review large` no longer widens the review, and `default-swarm-size` no longer affects
code review. Settled by the recorded boundary ("replace the current sizing options for code reviews") and the
confirmation turn.

**Why.** The operator's direction of travel.

**Migration.** Drop the size word. Use a type token to choose a focus.

**Decision.** [D-9](artifacts/change-decision-log.md#d-9-code-review-stops-reading-default-swarm-size), [D-19](artifacts/change-decision-log.md#trivial-decisions)

### S-4: Severity calibration in the Step 3.3 directive — Re-scoped

**Target state.** The Step 3.3 calibration directive carries one severity rule for every run: Critical and Warning
findings escalate, and a Suggestion is raised only when this change directly introduces it. It remains the single
authoritative home for severity calibration. The Review Constraints preamble, the Mode B and Mode C note, the Step 7.3
heading, the validator brief in `finding-filters.md`, the YAGNI wording in `review-checklist.md`, and Step 10 point at
it without naming a size.

**Behavior.** Changing. Former Small runs gain Suggestions the change introduced. Former Large runs lose pre-existing
Suggestions. Manual findings no longer prefer the higher severity at any size. The operator chose this rule.

**Why.** Size is gone, so one rule has to replace the ladder.

**Depends on.** S-3.

**Decision.** [D-8](artifacts/change-decision-log.md#d-8-one-severity-rule-replaces-the-size-ladder)

### S-5: Argument grammar (frontmatter and Step 1 parse) — Re-scoped

**Target state.** The frontmatter declares `arguments: type` and the `argument-hint` given in Target State. Step 1
parses the raw argument string by the pinned grammar into the type, the type's required input, `$focus_areas`, and,
in Mode C, the file list.

**Behavior.** Preserving for every invocation whose first token is neither a type token nor a size word. Size words are
covered by S-3.

**Why.** One positional slot served three readings with no stated rule.

**Depends on.** S-3.

**Decision.** [D-7](artifacts/change-decision-log.md#d-7-the-argument-grammar)

### S-6: `## Review Types` section in `SKILL.md` — Added

**Target state.** `SKILL.md` has a `## Review Types` section holding the type table (token, link, purpose), the default
set, and the default always-run agents. It is the only list of type tokens and the only link to each type file.

**Behavior.** Preserving (new structure).

**Why.** The plugin-builder guidance requires a direct link from `SKILL.md` to every reference file, and the default
run needs a named set.

**Decision.** [D-2](artifacts/change-decision-log.md#d-2-skillmd-holds-the-only-list-of-review-types-type-files-carry-no-links), [D-6](artifacts/change-decision-log.md#d-6-no-type-means-one-run-of-the-default-set-with-todays-roster-rule)

### S-7: The type-file contract — Added

**Target state.** Every file in `references/types/` has a `# Review type: {token}` title and the six sections Purpose,
Required input, Roster, Checklist categories, Instructions, and Report section, in that order, each holding a value of
the pinned form or `none`. Step 3.1 stops with `Review type file {path} is missing section "{name}"` when one is absent.

**Behavior.** Preserving (new structure).

**Why.** `SKILL.md` and every type file must agree on these fields independently.

**Depends on.** S-6.

**Decision.** [D-3](artifacts/change-decision-log.md#d-3-the-type-file-contract-is-six-fixed-sections)

### S-8: `references/types/security.md` — Added

**Target state.** The `security` type always runs `han-core:adversarial-security-analyst`, adds no other agent, and
sweeps Data Isolation, Error Handling, and API Design by hand. It is the only focused type that runs the security
check.

**Behavior.** Changing, as a set with S-9 to S-12: a run with any other type token runs no security check. The operator
decided this ("'security' will be it's own type, and will not automatically run when calling other types").

**Why.** The operator's answer to the security question.

**Depends on.** S-7.

**Decision.** [D-4](artifacts/change-decision-log.md#d-4-the-first-set-of-types), [D-5](artifacts/change-decision-log.md#d-5-security-is-its-own-type-and-no-type-points-at-another)

### S-9: `references/types/testing.md` — Added

**Target state.** The `testing` type always runs `han-core:test-engineer` and `han-core:edge-case-explorer` and sweeps
Testing and Correctness by hand.

**Behavior.** Preserving (new type).

**Why.** One of the operator's named types.

**Depends on.** S-7.

**Decision.** [D-4](artifacts/change-decision-log.md#d-4-the-first-set-of-types)

### S-10: `references/types/architecture.md` — Added

**Target state.** The `architecture` type always runs `han-core:structural-analyst` and `han-core:behavioral-analyst`,
adds `han-core:concurrency-analyst` and `han-core:data-engineer` when their signals fire, and sweeps Code Organization,
API Design, Database, and Performance by hand.

**Behavior.** Preserving (new type).

**Why.** One of the operator's named types.

**Depends on.** S-7.

**Decision.** [D-4](artifacts/change-decision-log.md#d-4-the-first-set-of-types)

### S-11: `references/types/observability.md` — Added

**Target state.** The `observability` type always runs `han-core:observability-engineer`, adds
`han-core:on-call-engineer` and `han-core:devops-engineer` when their signals fire, and sweeps Error Handling by hand.

**Behavior.** Preserving (new type).

**Why.** One of the operator's named types.

**Depends on.** S-7.

**Decision.** [D-4](artifacts/change-decision-log.md#d-4-the-first-set-of-types)

### S-12: `references/types/maintainability.md` — Added

**Target state.** The `maintainability` type always runs `han-core:junior-developer` and sweeps Code Maintainability,
Documentation, Code Style & Patterns, Architecture Decision Records, and Packaging by hand.

**Behavior.** Preserving (new type).

**Why.** Gives the generalist reviewer a home, so the default set covers today's whole roster.

**Depends on.** S-7.

**Decision.** [D-4](artifacts/change-decision-log.md#d-4-the-first-set-of-types)

### S-13: Step 3.2 agent selection — Re-scoped

**Target state.** Step 3.2 builds the roster from the resolved type: every agent on its `Always` line, plus every agent
on its `When relevant` line whose roster-table signal fires. For the default run it takes the union of the default
set's agents under the default rule in `## Review Types`. Extra Agents from configuration and agents the user named
explicitly join as they do today. Step 3.2 no longer carries an always-dispatch list of its own.

**Behavior.** Preserving for the default run. Changing for typed runs, per S-8.

**Why.** Which agents run is now a property of the type.

**Depends on.** S-1, S-6, S-8 to S-12.

**Decision.** [D-5](artifacts/change-decision-log.md#d-5-security-is-its-own-type-and-no-type-points-at-another), [D-6](artifacts/change-decision-log.md#d-6-no-type-means-one-run-of-the-default-set-with-todays-roster-rule)

### S-14: Step 3 title and Step 3.1 — Re-scoped

**Target state.** Step 3 is "Resolve the Review Type and Dispatch Review Agents". Step 3.1 resolves the type from the
Step 1 parse, reads its type file, stops on a missing section, and prints the review-type line. Steps 3.2 to 3.5 keep
their numbers.

**Behavior.** Changing only in what is announced: the run states its type instead of a size band.

**Why.** Step 3.1 was the size classifier.

**Depends on.** S-5, S-7.

**Decision.** [D-6](artifacts/change-decision-log.md#d-6-no-type-means-one-run-of-the-default-set-with-todays-roster-rule), [D-17](artifacts/change-decision-log.md#trivial-decisions)

### S-15: Run naming, from Step 8.6 to a new Step 3.0 — Moved

**Target state.** Step 3.0 resolves the slug and `{run_name}` before dispatch, by the same slug rules Step 8.6 used,
plus a type's `slug:` rule when it has one. Step 8.6 writes the report under `{run_name}.md` and no longer derives the
slug.

**Behavior.** Preserving for the slug itself. The suffix is S-16's change.

**Why.** The working folder has to be named before agents write into it.

**Decision.** [D-10](artifacts/change-decision-log.md#d-10-each-run-gets-its-own-working-folder-and-a-typed-report-name)

### S-16: Agent working files and the report name — Re-scoped

**Target state.** Every agent writes into `{run_directory}`, a folder named for the run. A typed run's report is named
`code-review-{slug}-{type}.md`; the default run's report keeps `code-review-{slug}.md`. Each selected agent's file holds
a pending marker until the agent overwrites it, and Step 7.1 never reads a file still holding the marker as findings.

**Behavior.** Changing. Agent working files move from `docs/{name}.md` into a per-run folder, and typed reports carry a
suffix. Settled by the recorded boundary ("so reviews can be parallelized").

**Why.** Fixed names collide across parallel runs and can be read stale.

**Depends on.** S-1, S-15.

**Migration.** Look for agent output under `docs/code-review-{slug}[-{type}]/` instead of `docs/`.

**Decision.** [D-10](artifacts/change-decision-log.md#d-10-each-run-gets-its-own-working-folder-and-a-typed-report-name)

### S-17: Step 1.6, ticket loading — Added

**Target state.** Step 1.6 runs only for `work-ticket`, in every git mode. It reads a local file with `Read` and fetches
any other reference with whatever tooling the installation provides, naming no tool. On failure it prints the pinned
message and waits for the operator's ticket text or instructions, then continues. `allowed-tools` is unchanged.

**Behavior.** Preserving (only `work-ticket` has required input).

**Why.** The type cannot do its job without the ticket, and the operator does not know in advance how each ticket is fetched.

**Depends on.** S-5, S-15.

**Decision.** [D-11](artifacts/change-decision-log.md#d-11-the-skill-loads-the-ticket-with-whatever-tooling-the-installation-has-and-waits-on-the-user-when-it-cannot)

### S-18: `han-core:gap-analyzer` in the shared roster — Added

**Target state.** `gap-analyzer` has a roster-table row (output `ticket-gap-analysis.md`, series `GAP`), a prompt, a
rubric section, an exemption from the Step 7.2 reachability gate, and a manual-sweep row.

**Behavior.** Preserving (no existing type dispatches it).

**Why.** It already compares an implementation against a requirements document.

**Depends on.** S-1, S-17.

**Decision.** [D-12](artifacts/change-decision-log.md#d-12-gap-analyzer-joins-the-shared-roster)

### S-19: `references/types/work-ticket.md` — Added

**Target state.** The `work-ticket` type requires a ticket reference, always runs `han-core:gap-analyzer`, adds
`han-core:test-engineer` when its signal fires, sweeps Correctness and Testing by hand, briefs every agent to judge the
change against the ticket, and adds the Ticket Resolution section to the report. Its full text is the worked example in
Target State.

**Behavior.** Preserving (new type).

**Why.** One of the operator's named types, and the one the request opens with.

**Depends on.** S-7, S-17, S-18.

**Decision.** [D-4](artifacts/change-decision-log.md#d-4-the-first-set-of-types), [D-13](artifacts/change-decision-log.md#d-13-work-ticket-adds-a-ticket-resolution-section)

### S-20: `default-swarm-size` docs and ADR 0001 — Re-scoped

**Target state.** `docs/configuration.md` names seven sizing-aware skills and no longer mentions code review in the
`default-swarm-size` section. `docs/sizing.md` no longer lists code review. ADR 0001 carries an appended note that
code review moved to review types; its decision is unchanged. `config-rule.md` and its vendored copies are unchanged.

**Behavior.** Preserving (documentation of S-3).

**Why.** Three docs name code review as a consumer of the setting.

**Depends on.** S-3.

**Decision.** [D-9](artifacts/change-decision-log.md#d-9-code-review-stops-reading-default-swarm-size)

### S-21: Report template — Re-scoped

**Target state.** `references/template.md` carries a `**Review type:**` header line and one slot after Review
Recommendation that renders a type's `## Report section` when it is not `none`. Every other section, and their order,
is unchanged.

**Behavior.** Changing in one line: every report names its type, or `default`. Settled by D-18; the line names only
the run's own type.

**Why.** Two posted reviews of one PR need to be told apart, and `work-ticket` needs its verdict section.

**Depends on.** S-7.

**Decision.** [D-13](artifacts/change-decision-log.md#d-13-work-ticket-adds-a-ticket-resolution-section), [D-18](artifacts/change-decision-log.md#trivial-decisions)

### S-22: `han-github/skills/post-code-review-to-pr/SKILL.md` — Re-scoped

**Target state.** The PR-posting skill passes its argument string to `/code-review` verbatim, so a leading type token
selects that type. Its `argument-hint` carries the same type clause. Its posted body includes Ticket Resolution when
present. When `/code-review` ends without naming a report path, it relays that message and stops.

**Behavior.** Changing for one case: free text that begins with a type token now runs that type. Settled by the recorded
boundary (Operator-Stated Scope item 1).

**Why.** The operator wants types available when posting to a PR.

**Depends on.** S-5, S-19, S-21.

**Decision.** [D-14](artifacts/change-decision-log.md#d-14-post-code-review-to-pr-passes-the-argument-string-through-and-carries-ticket-resolution)

### S-23: Step 4 manual review scope — Re-scoped

**Target state.** Step 4 applies the type's checklist categories, or the union of the default set's for the default
run, and always runs the YAGNI pass. The type's `## Instructions` apply to Step 4 as they do to the agents. Steps 2, 5,
and 6 are unchanged.

**Behavior.** Preserving for the default run, whose union covers every category. New behavior for typed runs.

**Why.** A focused review should be focused in its manual pass too.

**Depends on.** S-7.

**Decision.** [D-15](artifacts/change-decision-log.md#d-15-step-4-is-scoped-by-type-steps-2-5-and-6-run-on-every-review)

### S-24: `scripts/review-types.bats` — Added

**Target state.** A Bats file beside `detect-review-context.bats` checks four agreements: every type file and its table
row and hint token; every type file's title and six sections in order; every rostered agent's roster-table row and
prompt; and the default set's checklist union against `review-checklist.md`.

**Behavior.** Preserving (test only).

**Why.** These agreements span files and have already drifted once.

**Depends on.** S-1, S-6, S-7, S-8 to S-12.

**Decision.** [D-16](artifacts/change-decision-log.md#d-16-a-bats-check-holds-the-type-contract-together)

### S-25: Manual-only mode — Re-scoped

**Target state.** The manual-only disclosure says "Coverage absent: every agent the {type or default} review
selected", and the sweep rows carry no size. The absent-coverage list is still whatever Step 3.2 selected.

**Behavior.** Preserving. The disclosure names the same agents.

**Why.** The disclosure text hard-codes today's always-run pair.

**Depends on.** S-4, S-13.

**Decision.** [D-1](artifacts/change-decision-log.md#d-1-type-files-name-agents-everything-about-briefing-an-agent-stays-shared-and-keyed-by-agent), [D-8](artifacts/change-decision-log.md#d-8-one-severity-rule-replaces-the-size-ladder)

### S-26: Operator docs for code review and the PR skill — Re-scoped

**Target state.** `han-coding/docs/skills/code-review.md` documents review types, the default set, the single severity
rule, the run folder, and `work-ticket`, and no longer documents sizes. `han-coding/README.md`, `docs/skills/README.md`,
and `docs/choosing-a-han-plugin.md` reuse its new summary line. `han-github/docs/skills/post-code-review-to-pr.md` no
longer describes a size-aware roster. `CHANGELOG.md` records every Behavior Changes entry.

**Behavior.** Preserving (documentation).

**Why.** Every doc site in C-22 describes sizes.

**Depends on.** S-3 to S-25.

**Decision.** [D-4](artifacts/change-decision-log.md#d-4-the-first-set-of-types), [D-8](artifacts/change-decision-log.md#d-8-one-severity-rule-replaces-the-size-ladder), [D-10](artifacts/change-decision-log.md#d-10-each-run-gets-its-own-working-folder-and-a-typed-report-name)

## Behavior Changes

1. **Size words do nothing.** `/code-review large` runs the default review and prints a note that the word was ignored.
   Anyone who typed a size sees this. Decided by your request to replace sizing (S-3).
2. **The project's default size setting stops affecting code review.** A project with `default-swarm-size: large` used
   to get every review at Large; now it gets the single rule, silently. Other skills still read the setting. Decided by
   your request and the confirmed scope (S-3, S-20).
3. **Minor suggestions change.** A review of one to three files now lists suggestions the change introduced, where it
   used to hide them. A review of more than ten files no longer lists old suggestions the change did not cause. PR
   reviews still request changes on exactly the same findings. You chose this rule (S-4).
4. **A focused review skips security, without saying so.** `/code-review testing` runs no security check, and its
   report does not mention that. A bare `/code-review` still runs security. You decided this (S-8, S-13).
5. **Agent working files move into a folder per run, and typed reports get a suffix.** For example
   `docs/security-analysis.md` becomes `docs/code-review-my-branch/security-analysis.md`, and a testing review's report
   becomes `docs/code-review-my-branch-testing.md`. A bare review's report keeps its name. Decided by your goal of
   parallel reviews (S-16).
6. **Every report names its type.** A one-line header says which review type produced it (S-21).
7. **Posting to a PR can start a typed review.** If the text after `/post-code-review-to-pr` begins with a type name,
   such as "testing the retry logic", it now runs a `testing` review. Decided by your confirmed scope (S-22).

## Change Units

Units 2 through 7 should land on one branch before a release, because the docs in Unit 7 describe the finished state.
Each unit still leaves the skill runnable and `npm test` passing.

### Unit 1: Consolidate the roster table

**What it does.** Merges the signal and slice tables into one roster table with output-file and series columns, and
points Step 7.1 and the verification check at it. Nothing a user sees changes.

**Delta entries.** S-1, S-2.

**How you know it worked.** A default review on a sample branch dispatches the same agents and reads the same files as
before; the verification check now names all 11 agents.

### Unit 2: Remove size and adopt the single severity rule

**What it does.** Deletes the size classifier and every size reading, adds the retired-word note, installs the single
severity rule, and updates the sizing and configuration docs and ADR 0001. The default roster is still today's,
selected inside Step 3.2.

**Delta entries.** S-3, S-4, S-20, and the size wording in S-25.

**Ordering constraint.** After Unit 1, so the rewritten Step 3.3 and Step 7.1 land on the consolidated table.

**How you know it worked.** A search for `{size}`, `$size`, and `default-swarm-size` under `han-coding/skills/code-review/`
finds nothing; `/code-review large` prints the note and runs.

### Unit 3: Give each run its own folder

**What it does.** Moves slug resolution to Step 3.0, writes agent files into `code-review-{slug}/`, and adds the pending
marker. Reports keep today's name, because no type exists yet.

**Delta entries.** S-15, S-16 (without the type suffix).

**Ordering constraint.** After Unit 1, which owns the output-file column.

**How you know it worked.** Two default reviews of different branches run at once and leave two separate folders; an
agent that fails to write shows as not covered rather than as stale findings.

### Unit 4: Add the type mechanism and the five default-set types

**What it does.** Adds `## Review Types`, the type-file contract, the five type files, the argument grammar, type-based
Step 3.1 and 3.2, the default set, Step 4 scoping, the report's type line, the type suffix on run names, the generic
manual-only line, and the Bats check.

**Delta entries.** S-5 to S-14, S-16 (type suffix), S-21 (type line), S-23, S-24, S-25.

**Ordering constraint.** After Units 2 and 3. The five types land together because the default set needs all five to
reproduce today's roster, and Bats check 4 fails until they do.

**How you know it worked.** `npm test` passes, including `review-types.bats`. A bare `/code-review` dispatches the same
agents as before Unit 4. `/code-review testing` dispatches only the test and edge-case agents and writes
`code-review-{slug}-testing.md`.

### Unit 5: Add `work-ticket`

**What it does.** Adds Step 1.6, the `gap-analyzer` rows, the Step 7.2 exemption, the template's
report-section slot, and the `work-ticket` type file.

**Delta entries.** S-17, S-18, S-19, S-21 (report-section slot).

**Ordering constraint.** After Unit 4.

**How you know it worked.** `/code-review work-ticket docs/tickets/x.md` on a branch that meets half the ticket reports
"Partially resolved" with a row per requirement; `/code-review work-ticket missing.md` prints the pinned message and waits; pasting the ticket's
text continues the review.

### Unit 6: Pass types through `post-code-review-to-pr`

**What it does.** Passes the argument string through verbatim, matches the argument hint, carries Ticket Resolution,
and stops cleanly when code review stops.

**Delta entries.** S-22.

**Ordering constraint.** After Units 4 and 5.

**How you know it worked.** `/post-code-review-to-pr testing` drafts a body from a `-testing` report; a `work-ticket`
run with a bad path waits for the ticket inside the PR skill's run, and posts once the review completes.

### Unit 7: Update the operator docs

**What it does.** Rewrites the long-form code-review doc, the scent lines that reuse its summary, the PR skill's doc,
and the CHANGELOG entry listing every Behavior Changes item.

**Delta entries.** S-26.

**Ordering constraint.** Last.

**How you know it worked.** A search for "size-aware", "size-scaled", and `/code-review medium` across `han-coding/`,
`han-github/`, and `docs/` finds nothing outside `docs/plans/` and `CHANGELOG.md` history.

## Risks

- **A focus word that matches a type.** `/code-review testing the retry path` used to be a broad review focused on
  testing; it becomes a `testing` review. The Step 3.1 announcement shows this before dispatch. Blast radius: one run.
- **Unit 4 is large.** It touches `SKILL.md`, `agent-dispatch.md`, the template, and six new files at once. The Bats
  check and a before-and-after comparison of the default roster are the early signals.
- **A fetch that returns a sign-in page.** A tracker that requires sign-in can return a login page, and the "no ticket
  content" check is a judgment the model makes. A false pass would verify the change against a login page. The
  `ticket.md` copy in the run folder lets the operator see what was checked, and the Ticket Resolution table lists the
  requirements it found.
- **The type-file contract is read by a model, not a parser.** A malformed file could be read loosely instead of
  stopping. The Bats check catches structure; it cannot catch a misread value.
- **Silent loss of security on focused runs** is accepted by the operator, and is listed here so a reviewer of the
  plan sees it.

## Deferred (YAGNI)

- **More types (`data`, `resilience`, and others).** Reopen at the operator's planned refinement after the first build.
- **User-defined types through configuration.** Not requested. Reopen when an operator asks, or edits a shipped type
  file locally.
- **A type registry or index file.** It would duplicate the `## Review Types` table. Reopen if `SKILL.md` can no longer
  link every type file.
- **A severity band per type.** Reopen when a type's output is measured as too noisy or too sparse under the single
  rule.
- **Several named types in one invocation.** Parallel runs deliver this, and the default already combines types. Reopen
  when an operator records friction from launching runs separately.
- **A configurable default type.** Not requested. Reopen on request.
- **Scoping Steps 2, 5, and 6 by type.** Reopen when duplicate automated-check or doc findings across parallel typed
  reports are reported as friction.
- **Naming a fetch method per tracker.** The operator asked the skill to assume nothing about fetching. Reopen when
  operators ask for a named method with a second tracker in use.

## Cut for Scope

- **Hardening the branch-context markers against content that contains the end marker**
  ([C-9](artifacts/current-state-findings.md#c-9-untrusted-content-handling-is-explicit-and-reusable)). It would stop a
  PR body from closing the "untrusted" block early. The recorded scope covers replacing sizing with types, not
  hardening Step 1.5 (`artifacts/scope-boundary.md`, Stated Scope). You can reinstate it.

## Open Items

- **How the harness binds `arguments:`** (non-blocking). The grammar parses the raw string so the plan does not depend
  on it; a probe run during Unit 4 settles it.

## Review Findings

Pending the Step 7 review round.
