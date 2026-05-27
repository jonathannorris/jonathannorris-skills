## Extended sections (use when appropriate)

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
