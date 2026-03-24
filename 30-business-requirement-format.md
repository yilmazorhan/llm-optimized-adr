# Issue: Standardized Business Requirement Document Format

Unstructured business requirements cause ambiguity, leading to incorrect implementations. A standard, actionable format for Business Requirement Documents (BRDs) is needed to ensure every requirement is traceable, testable, and consistently structured across the project.

# Decision:

All Business Requirement Documents **MUST** be stored as versioned Markdown files in the `brd/` directory, following the `BR-NNN-descriptive-name.md` naming convention. Each BRD **MUST** contain the following numbered sections:

1. **Summary** — A high-level overview including a user story in "As a [role], I want [goal], So that [benefit]" format and a prose description of the feature scope.
2. **Business Rules** — An itemized list of rules using the BRD identifier prefix (e.g., `BR-001-01`, `BR-001-02`), each using RFC 2119 requirement levels (SHALL, MUST, SHOULD).
3. **Acceptance Criteria / Scenarios** — Detailed scenarios in Gherkin format (`Given`/`When`/`Then`) to make requirements concrete and directly translatable to automated tests.
4. **Cross-Cutting Concerns Checklist** — A table mapping each non-functional concern (security, logging, i18n, observability, etc.) to its governing ADR reference, planned implementation detail, and verification status.
5. **Related ADRs** — Explicit links to all architectural decisions that govern the feature implementation.

Optional sections **MAY** include: Scope (In-Scope / Out-of-Scope), Objectives, Non-Functional Requirements, Implementation Phases, REST API Endpoints, and Database Schema.

# Constraints:

- BRD files **MUST** reside in the `brd/` directory at the repository root
- BRD filenames **MUST** follow `BR-NNN-descriptive-name.md` where NNN is a zero-padded sequence number
- Business rules **MUST** use RFC 2119 requirement levels (MUST, SHALL, SHOULD, MAY)
- Acceptance Criteria **MUST** use Gherkin syntax (`Given`/`When`/`Then`)
- Cross-Cutting Concerns Checklist **MUST** reference governing ADRs by number
- BRD identifiers **MUST** be referenced in corresponding source code and test files for traceability

# Alternatives:

**Confluence / Wiki-Based Requirements**: Atlassian Confluence documentation (https://support.atlassian.com/confluence-cloud/docs/create-and-edit-pages/) stores requirements as web pages with rich formatting. Requirements stored in a wiki are decoupled from the codebase — Confluence pages have no native mechanism to enforce co-versioning with Git branches, meaning requirement text can diverge from the code it describes after any merge. Confluence search returns pages but does not support `grep`-based traceability from source code back to requirement identifiers.

**Issue Tracker Requirements (Jira Epics/Stories)**: Jira documentation (https://support.atlassian.com/jira-software-cloud/docs/what-is-an-issue/) represents requirements as issues with fields and workflow states. Jira issues use a separate numbering system (project key + auto-increment) that is not portable to other tools, and the Gherkin-formatted acceptance criteria must be stored in description fields that lack syntax highlighting or structural validation. The Jira REST API (https://developer.atlassian.com/cloud/jira/platform/rest/v3/) supports export, but extracted Markdown lacks the numbered-section structure needed for automated compliance checking.

# Rationale:

The Gherkin syntax specification (https://cucumber.io/docs/gherkin/reference/) defines a structured `Given`/`When`/`Then` format that maps directly to executable test steps in Cucumber, Karate, and similar frameworks — every acceptance scenario written in Gherkin is parseable by test runners without manual translation. Storing BRDs as Markdown files in the same Git repository as source code ensures atomic commits: a feature branch contains both the requirement changes and the implementing code, which `git log --all -- brd/BR-001*.md` can trace. The RFC 2119 keyword convention (https://www.rfc-editor.org/rfc/rfc2119) provides unambiguous requirement levels (MUST vs. SHOULD vs. MAY) that eliminate interpretation disputes across teams. The Cross-Cutting Concerns Checklist links each non-functional area to its governing ADR, producing a verifiable mapping rather than implicit assumptions.

# Implementation Guidelines:

## BRD Template

Each new BRD **MUST** begin from the following structure:

```markdown
# BR-NNN: Title

## 1. Summary
[User story in "As a [role], I want [goal], So that [benefit]" format]

[Prose description of feature scope]

## 2. Business Rules
- **BR-NNN-01**: [Rule using RFC 2119 levels]
- **BR-NNN-02**: [Rule using RFC 2119 levels]

## 3. Acceptance Criteria / Scenarios
\```gherkin
Scenario: [Descriptive scenario name]
  Given [precondition]
  When [action]
  Then [expected outcome]
\```

## 4. Cross-Cutting Concerns Checklist
| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Security | ADR-18 | [Details] | TBD |
| Logging | ADR-16 | [Details] | TBD |
| i18n | ADR-25, ADR-26 | [Details] | TBD |
| Observability | ADR-20, ADR-21 | [Details] | TBD |

## 5. Related ADRs
- ADR-NN: [Title]
```

## Naming Convention

- Directory: `brd/`
- File pattern: `BR-NNN-descriptive-name.md`
- Rule identifiers: `BR-NNN-RR` (NNN = BRD number, RR = rule sequence)
- Sequence numbers: zero-padded three digits, starting from `000`

## Traceability Rules

Source code and test files **MUST** reference the governing BRD identifier in comments or annotations:

```java
// Implements BR-001-02: Audit events persisted to ClickHouse
```

```java
@DisplayName("BR-001-03: Query audit events by time range")
```

# Additional Recommendations:

- Use a BRD review checklist before merging: verify all 5 mandatory sections exist, all rules use RFC 2119 levels, all scenarios use Gherkin syntax, and the Cross-Cutting Concerns Checklist references valid ADR numbers
- Automate Gherkin scenario extraction from BRDs into test feature files to maintain requirement-to-test traceability
- Tag Git commits with BRD identifiers (e.g., `BR-001`) in commit messages for `git log` filtering
- Review BRDs when referenced ADRs are updated to ensure consistency

**References**:
- Gherkin Reference: https://cucumber.io/docs/gherkin/reference/
- RFC 2119 Requirement Levels: https://www.rfc-editor.org/rfc/rfc2119
- Markdown Guide: https://www.markdownguide.org/
- Cucumber BDD Documentation: https://cucumber.io/docs/bdd/
- Git Log Filtering: https://git-scm.com/docs/git-log

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| BRD Directory | `ls -d brd/` | Directory exists, exit code 0 |
| BRD Naming Convention | `find brd/ -name "BR-*.md" -type f \| head -5` | At least one BRD file matches `BR-NNN-*.md` pattern |
| Summary with User Story | `grep -rl "As a\|As an" brd/*.md \| head -5` | User story format present in BRD files |
| Business Rules with RFC 2119 | `grep -rl "SHALL\|MUST\|SHOULD" brd/*.md \| head -5` | RFC 2119 keywords in business rules |
| Gherkin Acceptance Criteria | `grep -rl "Given.*\|When.*\|Then.*" brd/*.md \| head -5` | Gherkin scenarios present |
| Cross-Cutting Concerns | `grep -rl "Cross-Cutting Concerns\|ADR Reference" brd/*.md \| head -5` | Checklist table present |
| BRD Traceability in Code | `grep -rl "BR-[0-9]" */src/ 2>/dev/null \| head -5` | BRD identifiers referenced in source/test files |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
echo "Validating Business Requirement Format..."

FAIL=0

# Check BRD directory exists
if [ -d "brd" ]; then
  echo "✅ BRD directory exists"
else
  echo "❌ BRD directory not found"
  FAIL=1
fi

# Check BRD naming convention
FOUND=0
for f in brd/BR-*.md; do
  if [ -f "$f" ]; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  BRD_COUNT=$(find brd/ -name "BR-*.md" -type f 2>/dev/null | wc -l | tr -d ' ')
  echo "✅ BRDs follow BR-NNN naming convention ($BRD_COUNT files)"
else
  echo "❌ No BRD files matching BR-*.md pattern"
  FAIL=1
fi

# Check for user story format
FOUND=0
for f in brd/BR-*.md; do
  if [ -f "$f" ] && grep -ql "As a\|As an" "$f" 2>/dev/null; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ User story format detected in BRDs"
else
  echo "❌ No user story format found in BRD files"
  FAIL=1
fi

# Check for RFC 2119 keywords in business rules
FOUND=0
for f in brd/BR-*.md; do
  if [ -f "$f" ] && grep -ql "SHALL\|MUST\|SHOULD" "$f" 2>/dev/null; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ RFC 2119 requirement levels used in business rules"
else
  echo "❌ No RFC 2119 keywords (SHALL/MUST/SHOULD) in BRD files"
  FAIL=1
fi

# Check for Gherkin acceptance criteria
FOUND=0
for f in brd/BR-*.md; do
  if [ -f "$f" ] && grep -ql "Given\|When\|Then" "$f" 2>/dev/null; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Gherkin acceptance criteria found"
else
  echo "❌ No Gherkin scenarios (Given/When/Then) in BRD files"
  FAIL=1
fi

# Check for Cross-Cutting Concerns checklist
FOUND=0
for f in brd/BR-*.md; do
  if [ -f "$f" ] && grep -ql "Cross-Cutting Concerns\|ADR Reference" "$f" 2>/dev/null; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Cross-Cutting Concerns checklist present"
else
  echo "❌ No Cross-Cutting Concerns checklist in BRD files"
  FAIL=1
fi

# Check for BRD traceability in source code
FOUND=0
for d in */src; do
  if [ -d "$d" ] && grep -rql "BR-[0-9]" "$d" 2>/dev/null; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ BRD traceability references found in source code"
else
  echo "❌ No BRD identifier references (BR-NNN) in source/test files"
  FAIL=1
fi

# Build verification
if ./mvnw clean verify -q 2>/dev/null; then
  echo "✅ Build verification passed"
else
  echo "❌ Build verification failed"
  FAIL=1
fi

if [ "$FAIL" -ne 0 ]; then
  echo "❌ Business Requirement Format validation failed"
  exit 1
fi

echo "✅ All Business Requirement Format criteria met"
```
