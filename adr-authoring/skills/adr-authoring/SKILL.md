---
name: adr-authoring
description: Author Architecture Decision Records (ADRs) from proposals, issues, or scratch. Covers research, drafting in MADR format, numbering, iteration based on PR feedback, and syncing related spec or schema changes. Use when asked to write, draft, update, or iterate on an ADR — e.g. "help me write an ADR for X", "turn this proposal into an ADR", "create a new ADR based on this issue", "update the ADR based on the PR feedback".
---

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

- **Source material:** GitHub issue/PR (`gh issue view` / `gh pr view`), PDF, gist (`curl` / `gh gist view`), or verbal -- ask clarifying questions if scope is unclear
- **Codebase:** Read related ADRs this decision builds on or supersedes; check spec files (OpenAPI, protocol definitions) that the decision affects
- **External:** Research referenced patterns (OpenTelemetry, vendor SDKs, etc.) to inform rationale and alternatives

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

For extended section templates (structured consequences, alternatives considered, implementation notes, open questions), see [REFERENCE.md](REFERENCE.md).

### Diagrams and code blocks

- Use **Mermaid** sequence or flow diagrams to illustrate complex interactions (e.g., client-server flows, initialization sequences)
- Use **JSON** or **YAML** code blocks for schema definitions, API payloads, or configuration examples
- Verify Mermaid diagrams render correctly in GitHub by checking the preview

For PR iteration, related spec changes, commit format, and OpenFeature-specific hints, see [REFERENCE.md](REFERENCE.md).
