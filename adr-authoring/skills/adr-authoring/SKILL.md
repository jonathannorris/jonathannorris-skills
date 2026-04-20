---
name: adr-authoring
description: Author Architecture Decision Records (ADRs) from proposals, issues, or scratch. Covers research, drafting in MADR format, numbering, iteration based on PR feedback, and syncing related spec or schema changes.
---

## When to use this skill

Use this skill when asked to create, draft, or update an Architecture Decision Record. This includes requests like:
- "help me write an ADR for X"
- "turn this proposal into an ADR"
- "create a new ADR based on this issue / PDF / gist"
- "update the ADR based on the PR feedback"
- "draft an ADR for adding feature X to the protocol"

## Step 1: Understand the project's ADR setup

### Locate the ADR directory

Check for an `.adr-dir` file in the repo root. If it exists, it contains the path to the ADR directory (e.g., `./service/adrs`). Otherwise, look for common locations:

```bash
# Check for adr-tools config
cat .adr-dir 2>/dev/null

# Common ADR directory locations
ls -d service/adrs/ docs/adr/ docs/adrs/ adr/ adrs/ decisions/ 2>/dev/null
```

### Study existing ADR format

Read ALL existing ADRs in the directory to understand:
- Heading format (e.g., `# N. Title`)
- Which sections are used (`Status`, `Context`, `Decision`, `Consequences`, etc.)
- Whether extended sections appear (`Alternatives Considered`, `Implementation Notes`, `Open Questions`)
- Whether `Consequences` uses flat bullets or `### Positive` / `### Negative` subsections
- Use of code blocks, diagrams (Mermaid), JSON/YAML examples
- Date format
- Status values used (`Accepted`, `Proposed`, `Deprecated`, `Superseded`)

Match the format of the most recent ADRs, since style may have evolved over time.

### Determine the next ADR number

```bash
ls <adr-directory>/ | sort -n | tail -1
```

Increment the highest existing number by 1. Pad with leading zeros to match the existing convention (e.g., `0010` if others use 4-digit padding).

## Step 2: Research and gather context

Before drafting, gather relevant context depending on what the ADR covers:

### From source material

The user may provide starting material in various forms:
- **GitHub issue or PR URL** -- fetch with `gh issue view` or `gh pr view`
- **PDF or local file** -- read and extract the key proposal details
- **Gist URL** -- fetch with `curl` or `gh gist view`
- **Verbal description** -- ask clarifying questions if the scope is unclear

### From the codebase

- Read related existing ADRs that this decision builds on or supersedes
- Check any specification files (OpenAPI specs, protocol definitions, etc.) that the decision affects
- Look at related implementations in sibling repos if accessible (the user may point you to them)

### From external sources

- If the ADR references patterns from other projects (e.g., OpenTelemetry, vendor SDKs), research those patterns to inform the decision rationale and alternatives

## Step 3: Draft the ADR

### File naming

Use lowercase, hyphen-separated words with the ADR number prefix:

```
<number>-<short-descriptive-name>.md
```

Examples: `0008-sse-for-bulk-evaluation-changes.md`, `0009-local-storage-for-static-context-providers.md`

### Base structure (MADR)

Every ADR must include these sections:

```markdown
# <Number>. <Title>

Date: YYYY-MM-DD

## Status

Proposed

## Context

<Narrative explanation of the problem, situation, or opportunity. Include enough
background that a reader unfamiliar with the discussion can understand why a
decision is needed.>

## Decision

<What was decided and why. Be specific about the technical approach. Include
schema definitions, API changes, configuration options, or behavioral
specifications as appropriate.>

## Consequences

<What follows from this decision. What becomes easier, harder, or changes.>
```

### Extended sections (use when appropriate)

For complex decisions, add these sections as needed:

**Structured consequences** -- when trade-offs are significant:

```markdown
## Consequences

### Positive

- <benefit>
- <benefit>

### Negative

- <trade-off>
- <trade-off>
```

**Alternatives considered** -- when the decision involved choosing between approaches:

```markdown
## Alternatives Considered

### <Alternative 1 name>

<Description and why it was not chosen.>

### <Alternative 2 name>

<Description and why it was not chosen.>
```

**Implementation notes** -- when providers or implementors need specific guidance:

```markdown
## Implementation Notes

<Guidance for implementors: required behavior, edge cases, configuration
options, error handling, migration path, etc.>
```

**Open questions** -- for `Proposed` status ADRs with unresolved items:

```markdown
## Open Questions

1. <Question>
   - <Answer or "TBD">
```

### Diagrams and code blocks

- Use **Mermaid** sequence or flow diagrams to illustrate complex interactions (e.g., client-server flows, initialization sequences)
- Use **JSON** or **YAML** code blocks for schema definitions, API payloads, or configuration examples
- Verify Mermaid diagrams render correctly in GitHub by checking the preview

## Step 4: Review and iterate

### Initial review

After drafting, do a self-review:
- Does the Context explain the "why" clearly?
- Does the Decision section stand alone as a specification someone could implement from?
- Are consequences honest about trade-offs (not just listing positives)?
- Is the ADR consistent with the format of existing ADRs in the project?

### PR feedback loop

When the ADR is submitted as a PR and review comments come in:

1. Review the comments (use the `pr-review` skill if available for structured comment processing)
2. Draft changes to the ADR based on the feedback
3. Present the proposed changes to the user before committing
4. After user approval, commit and push

When updating the ADR based on feedback:
- Keep the same ADR number and file name unless the scope fundamentally changed
- Update the `Date` field if the decision substance changed significantly
- If a comment requires a non-trivial design change, consider adding or updating the `Alternatives Considered` section to document why the previous approach was abandoned

## Step 5: Related changes

ADR decisions often require corresponding changes elsewhere:

### Specification or schema files

If the ADR introduces protocol changes (new endpoints, new fields, new behavior), check whether the project has:
- An OpenAPI spec that needs updating
- Protocol definition files
- Provider guidelines or implementation guides

Note these as follow-up work. Do NOT make spec changes unless the user explicitly asks.

### Cross-references

If the new ADR supersedes or amends an existing ADR:
- Update the old ADR's `## Status` to `Superseded by [ADR-NNNN](link)` or `Amended by [ADR-NNNN](link)`
- Reference the old ADR in the new ADR's `## Context` section

## Step 6: Commit and push

IMPORTANT: Follow the repo's `AGENTS.md` rules. Do NOT commit or push unless the user explicitly requests it.

### Commit message format

Use conventional commit format:

```
docs: add ADR-<number> <short description>
```

For updates to an existing ADR:

```
docs: update ADR-<number> <what changed>
```

### Sign-off

If the repo requires signed commits (check `AGENTS.md`), use `git commit -s`.

## Hints for OpenFeature protocol repos

These hints apply when working in OpenFeature protocol or spec repositories:

- ADRs live in `service/adrs/`, configured via `.adr-dir`
- The OpenAPI spec is at `service/openapi.yaml`; provider guidelines are in `guideline/`
- PR titles must follow Conventional Commits (enforced by CI)
- Spectral linting runs on the OpenAPI spec; if you update it, validate locally first
- Related provider implementations may be in sibling directories (e.g., `../js-sdk-contrib`, `../kotlin-sdk`, `../swift-sdk`)
- Vendor SDK implementations (DevCycle, LaunchDarkly, Statsig, Eppo) are useful references for understanding real-world behavior that ADRs should account for
