# Issue: Code Comment Standards

Code comments are unregulated: business-rule implementations carry no traceability to their governing BRD (ADR-30) or ADR, complex algorithms ship with no explanation of their approach, and noise comments (restating code, commented-out code, untracked TODOs) accumulate. Readers cannot answer "why does this code exist" or "which requirement does this satisfy" without archaeology.

**Impact**: Lost business rationale, unreviewable complex logic, stale/misleading comments, and broken requirement-to-code traceability (ADR-30) during audits and maintenance.

# Decision:

Implement a three-class comment taxonomy — **Traceability Comments**, **Algorithm Explanations**, and **API Documentation** — with noise comments forbidden and PMD enforcement integrated into the existing build pipeline (ADR-12).

**Required Components**:
- **Traceability Comments**: Code implementing a business rule (BRD) or an ADR-mandated behavior **MUST** reference the governing document identifier and the specific part (rule sequence or section) the code relates to
- **Reference Grammar**: BRD references **MUST** use `BR-NNN-RR` rule identifiers or `BR-NNN §<section>`; ADR references **MUST** use `ADR-NN §<section-name>`
- **Algorithm Explanations**: Code containing a highly complex algorithm **MUST** carry a brief explanation comment covering approach, invariants/edge cases, and an authoritative source link when the algorithm is published
- **Objective Complexity Trigger**: An explanation **MUST** be present when the method meets any trigger in the Complexity Trigger Table below
- **API Documentation**: Public and protected members of library modules (ADR-09) **MUST** have Javadoc per Google Java Style §7.2 (ADR-12)
- **Noise Forbidden**: Comments restating the code, commented-out code, and TODO/FIXME without a tracking identifier **MUST NOT** appear
- **Language**: Comments **MUST** be written in English
- **Enforcement**: PMD `CommentRequired` and `CommentSize` rules **MUST** run in the ADR-12 `build/pmd-ruleset.xml`; traceability **MUST** be validated by the ADR-30 success criteria script

# Ownership

| Role | Person | Competencies |
|------|--------|-------------|
| **Responsible** | Senior Backend Developer | Comment taxonomy application (traceability/algorithm/API classes), BRD and ADR reference grammar (`BR-NNN-RR`, `ADR-NN §`), Javadoc authoring per Google Java Style §7 (ADR-12), PMD comment ruleset configuration, algorithm explanation authoring for complex logic (Mutiny chains, concurrency, bit manipulation) |
| **Approver** | Tech Lead / Software Architect | Comment policy scope decisions (self-documenting vs. mandated traceability), reference grammar governance, complexity trigger thresholds (cognitive complexity documentation zone), code-review gate enforcement for algorithm explanations |

# Constraints:

- Traceability references **MUST** use exact identifiers that resolve in the repository: `BR-NNN-RR` (ADR-30 rule sequence), `BR-NNN §<number>`, or `ADR-NN §<section-name>`; broken references **MUST NOT** pass review
- A traceability comment **MUST** state which part of the document the code relates to; bare document numbers without a rule or section pointer **MUST NOT** be used alone
- Algorithm explanations **MUST NOT** exceed 15 lines (brief by mandate) and **MUST** include: approach, invariants or edge cases, and a source link when the algorithm is published
- Comments **MUST NOT** contain credentials, tokens, PII, or customer data (ADR-16, ADR-18 redaction policy)
- Commented-out code **MUST NOT** be committed (version control provides history)
- `TODO`/`FIXME` **MUST** include a tracking identifier (issue/ticket number) in the same line; untracked TODOs **MUST NOT** pass review
- Comments **MUST** be updated in the same commit as the behavior they describe; stale comments **MUST** be treated as build-blocking review findings
- Javadoc on public/protected members of library modules (ADR-09) **MUST** pass Checkstyle/PMD documentation checks (ADR-12)
- Comments **MUST NOT** restate what the code obviously expresses (Google Java Style §7.3 exception for self-explanatory members)
- Traceability comments **MUST NOT** substitute for clear naming (ADR-11 naming conventions remain primary)

# Alternatives:

1. **Self-Documenting Code Only (no comments policy)**: Rely exclusively on naming and structure; forbid explanatory comments. Rejected because: ADR-30 already mandates BRD identifier references in source and test files for traceability — a no-comment policy directly contradicts an accepted ADR (see ADR-30 Traceability Rules). Code cannot express *why* a business rule exists or which requirement governs a boundary value; that context lives only in the BRD. Google Java Style §7.2 requires Javadoc for public/protected members used outside their own module (https://google.github.io/styleguide/javaguide.html#s7.2), so zero-comment is also non-compliant with the ADR-12 style baseline.

2. **Mandatory Javadoc on Every Method (maximum comment density)**: Require a Javadoc block on every method, field, and class regardless of self-evidence. Rejected because: Google Java Style §7.3 states Javadoc is optional "for simple, obvious methods" such as `getFoo`, and that documentation adds no value when it merely restates the method name (https://google.github.io/styleguide/javaguide.html#s7.3). Uniform comment volume produces restatement noise that decays into stale comments — the specific failure mode this ADR forbids. The taxonomy approach mandates comments exactly where they carry non-derivable information (traceability, algorithms) and permits omission where code is self-evident.

# Rationale:

ADR-30 defines the BRD identifier grammar (`BR-NNN` documents, `BR-NNN-RR` rules) and already requires source/test references to governing identifiers; this ADR extends that convention with a precise in-code syntax that names the *part* of the document the code relates to, making the reference verifiable (`grep "BR-001-02"` in source maps to `requirements/BR-001-*.md` rule 02, per the ADR-30 naming convention). ADR documents are numbered and section-anchored (`# Decision:`, `# Constraints:`), so `ADR-NN §<section-name>` resolves unambiguously.

PMD provides source-level comment enforcement that runs in the existing ADR-12 pipeline: the `CommentRequired` rule family (class/method/field comment presence) and `CommentSize` rule (comment length limits) operate on Java source text during `pmd:check` (https://docs.pmd-code.org/latest/pmd_rules_java_comments.html). Google Java Style is the ADR-12 style baseline and defines both where Javadoc is required (§7.2) and where it is optional (§7.3), giving the taxonomy an authoritative, non-subjective boundary (https://google.github.io/styleguide/javaguide.html#s7-javadoc).

PMD's `CognitiveComplexity` rule quantifies how difficult a method is to read (nesting, breaks in linear flow, recursion), with the ADR-12 build limit set at 15 (https://docs.pmd-code.org/latest/pmd_rules_java_design.html#cognitivecomplexity). Using ≥10 (two-thirds of the failure limit) as the documentation trigger creates an objective, machine-checkable boundary for "highly complex" instead of reviewer opinion. Bit manipulation, concurrency ordering, and named algorithms (token bucket, sliding window, retry with jitter) are enumerated as additional triggers because cognitive complexity alone misses terse-but-cryptic code such as a one-line bit mask.

# Implementation Guidelines:

## 1. Comment Taxonomy

| Class | Required When | Format |
|-------|--------------|--------|
| **Traceability** | Code implements a business rule or ADR-mandated behavior | Line comment or Javadoc with reference grammar |
| **Algorithm Explanation** | Any Complexity Trigger Table row matches | Block comment before the method (template below) |
| **API Documentation** | Public/protected members of library modules (ADR-09, ADR-12 §7.2) | Javadoc |
| **Noise** | Never | Forbidden |

## 2. Traceability Reference Grammar

> [!IMPORTANT]
> Every traceability comment **MUST** name the document part (rule or section). Bare `BR-001` or `ADR-17` without a part pointer **MUST NOT** be used alone.

| Target | Syntax | Example |
|--------|--------|---------|
| BRD business rule | `Implements BR-NNN-RR: <intent>` | `// Implements BR-001-02: Audit events persisted to database` |
| BRD section | `BR-NNN §<number>: <intent>` | `// BR-002 §3 (Acceptance Criteria): dashboard export returns all UIDs` |
| ADR section | `ADR-NN §<section>: <intent>` | `// ADR-10 §Constraints: domain must not import adapter packages` |
| Multiple | Separate with `;` | `// Implements BR-001-02; ADR-19 §Decision: async audit sink` |
| Javadoc variant | Same grammar inside `@implNote` | `@implNote Implements BR-003-01: query endpoint paging` |

**Placement Rules**:
- Place the comment on the class when the whole class implements one rule; on the method when a single method does
- Test files **MUST** use `@DisplayName("BR-NNN-RR: ...")` for scenario naming (ADR-30, ADR-32)
- Traceability references **MUST NOT** be duplicated on every line of a block; one reference at the implementing unit

## 3. Algorithm Explanation Template

> [!IMPORTANT]
> **MUST** be ≤15 lines. **MUST** cover approach, invariants/edge cases, and source link (when published).

```java
/*
 * Algorithm: Token bucket rate limiter (ADR-18 rate limiting)
 * Approach: Fixed-capacity bucket; tokens refill at constant rate, are
 *   consumed per request; refill is computed lazily on read, not by timer.
 * Invariants: token count never exceeds capacity; timestamp monotonically
 *   increases; O(1) per request, memory bounded by one bucket per key.
 * Edge cases: clock skew handled by clamping elapsed time to >= 0.
 * Source: https://en.wikipedia.org/wiki/Token_bucket
 */
boolean tryAcquire(String key) { ... }
```

## 4. Complexity Trigger Table

An Algorithm Explanation **MUST** be present when **any** row matches:

| # | Trigger | Detection |
|---|---------|-----------|
| 1 | PMD CognitiveComplexity ≥ 10 (ADR-12 build limit is 15) | `./mvnw pmd:check` report |
| 2 | Bit manipulation (`<<`, `>>`, `>>>`, mask `&`/`|` on non-boolean) | Code review |
| 3 | Named/published algorithm or protocol (token bucket, sliding window, exponential backoff with jitter, consensus, graph traversal, crypto primitives) | Code review |
| 4 | Recursion with non-obvious invariants or termination conditions | Code review |
| 5 | Concurrency-order-sensitive logic (happens-before assumptions, lock ordering) | Code review |
| 6 | Performance optimization where the naive approach would observably differ (memoization, precomputation, batching) | Code review |

## 5. PMD Ruleset Extension (`build/pmd-ruleset.xml`, ADR-12)

```xml
<rule ref="category/java/comments.xml/CommentRequired">
    <properties>
        <property name="methodWithCommentRequirement" value="Required"/>
        <property name="fieldWithCommentRequirement" value="Ignored"/>
    </properties>
</rule>
<rule ref="category/java/comments.xml/CommentSize">
    <properties>
        <property name="maxLines" value="15"/>
        <property name="maxLineLength" value="120"/>
    </properties>
</rule>
```

> **Note**: `CommentRequired` applies to library modules' public API (ADR-09). Domain and application modules rely on the taxonomy + review gates rather than blanket method comment requirements, consistent with Google Java Style §7.3.

## 6. Noise Comment Examples (Forbidden)

```java
// ❌ i incremented
i++;

// ❌ get user by id (restates the method — ADR-11 naming carries this)
// ❌ commented-out code (git history owns old versions)
// userCache.clear();
// TODO fix this  (untracked — MUST carry an issue ID, e.g. TODO ISSUE-142: ...)
```

## 7. Validation Commands

```bash
# Comment rules via the ADR-12 pipeline
./mvnw pmd:check
./mvnw checkstyle:check

# Traceability spot-check (ADR-30 grammar)
grep -rn "Implements BR-[0-9]" */src/main/java | head

# Untracked TODO scan
grep -rn "TODO\|FIXME" */src/main/java | grep -v "ISSUE-" | wc -l   # MUST be 0
```

# Additional Recommendations:

- **RECOMMENDED** Prefer one high-value traceability comment per business-rule implementation over scattered references.
- **RECOMMENDED** Link the authoritative source (RFC, paper, spec) instead of paraphrasing published algorithms in full.
- **RECOMMENDED** Review comment accuracy during PR review with the same rigor as code; flag stale comments as blocking.
- **RECOMMENDED** When a method is complex enough to need an explanation *and* the explanation exceeds 15 lines, extract helpers until the explanation fits — complexity that cannot be briefly explained **SHOULD** be refactored (ADR-12 complexity limits).
- **MAY** use `@implNote`/`@implSpec` Javadoc tags for traceability in library modules to keep the information in generated Javadoc.
- **MUST** Reference ADR-09 (Multi-Module Structure), ADR-10 (Hexagonal Architecture), ADR-11 (Package Structure), ADR-12 (Code Quality Standards), ADR-16 (Structured Logging), ADR-17 (Exception Handling), ADR-18 (Security Implementation), ADR-30 (Business Requirement Format), ADR-32 (Unit Testing Strategy).
- See Google Java Style §7 (Javadoc): https://google.github.io/styleguide/javaguide.html#s7-javadoc
- See PMD Java Comments Rules: https://docs.pmd-code.org/latest/pmd_rules_java_comments.html
- See PMD CognitiveComplexity: https://docs.pmd-code.org/latest/pmd_rules_java_design.html#cognitivecomplexity

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| PMD Comment Rules Active | `grep CommentRequired build/pmd-ruleset.xml` | Rule present |
| Comment Size Limited | `grep CommentSize build/pmd-ruleset.xml` | `maxLines` ≤ 15 |
| BRD References Have Part Pointers | `grep -rEn "BR-[0-9]+($|[^0-9-§])" */src/main/java \| grep -v "BR-[0-9]+-[0-9]+\|BR-[0-9]+ §"` | Zero bare references |
| ADR References Have Sections | `grep -rEn "ADR-[0-9]+($|[^0-9 §])" */src/main/java` | All include `§` part pointer |
| No Untracked TODO/FIXME | `grep -rn "TODO\|FIXME" */src \| grep -cv "ISSUE-"` | Zero matches |
| No Commented-Out Code | Code review | Zero blocks |
| Algorithm Explanations on Triggers | `./mvnw pmd:check` + review | Every CognitiveComplexity ≥ 10 method has block comment |
| English-Only Comments | Review | No non-English comments |
| No Sensitive Data in Comments | `grep -rniE "password|secret|token|api[-_]?key" */src \| grep "/\*\|//"` | Zero real values |
| Build Passes | `./mvnw clean verify` | Exit code 0 (ADR-12) |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Code Comment Standards..."
FAIL=0

# Check PMD comment rules in the ADR-12 ruleset
for rule in CommentRequired CommentSize; do
  if grep -rq "$rule" build/pmd-ruleset.xml 2>/dev/null; then
    echo "✅ PMD $rule rule configured"
  else
    echo "❌ PMD $rule rule not found in build/pmd-ruleset.xml"
    FAIL=1
  fi
done

# BRD references must carry a part pointer (BR-NNN-RR or BR-NNN §)
BARE_BRD=$(grep -rEn "BR-[0-9]{3}" */src/main/java */src/test/java 2>/dev/null \
  | grep -v "BR-[0-9]{3}-[0-9]{2}" | grep -v "BR-[0-9]{3} §" | wc -l | tr -d ' ')
if [ "$BARE_BRD" -eq 0 ]; then
  echo "✅ All BRD references include rule/section pointers"
else
  echo "❌ Bare BRD references without part pointer: $BARE_BRD"
  FAIL=1
fi

# ADR references must carry a section pointer
BARE_ADR=$(grep -rEn "ADR-[0-9]{2}" */src/main/java */src/test/java 2>/dev/null \
  | grep -v "ADR-[0-9]{2} §" | wc -l | tr -d ' ')
if [ "$BARE_ADR" -eq 0 ]; then
  echo "✅ All ADR references include section pointers"
else
  echo "❌ Bare ADR references without section pointer: $BARE_ADR"
  FAIL=1
fi

# Untracked TODO/FIXME forbidden
UNTRACKED=$(grep -rn "TODO\|FIXME" */src 2>/dev/null | grep -v "ISSUE-" | wc -l | tr -d ' ')
if [ "$UNTRACKED" -eq 0 ]; then
  echo "✅ No untracked TODO/FIXME comments"
else
  echo "❌ Untracked TODO/FIXME found: $UNTRACKED occurrences"
  FAIL=1
fi

# Sensitive data in comments forbidden (ADR-16, ADR-18)
SENSITIVE=$(grep -rniE "(password|secret|token|apikey|api_key)\s*[:=]\s*[A-Za-z0-9_/+-]{8,}" */src 2>/dev/null \
  | wc -l | tr -d ' ')
if [ "$SENSITIVE" -eq 0 ]; then
  echo "✅ No sensitive values in comments"
else
  echo "❌ Potential sensitive data in comments: $SENSITIVE"
  FAIL=1
fi

# PMD comment checks pass (includes CognitiveComplexity report for review)
./mvnw pmd:check -q && echo "✅ PMD comment checks passed" || { echo "❌ PMD check failed"; FAIL=1; }

# Full build (ADR-12: no skips)
./mvnw clean verify -q && echo "✅ Full build verification passed" || { echo "❌ Build failed"; FAIL=1; }

if [ "$FAIL" -ne 0 ]; then
  echo "❌ Code Comment Standards validation FAILED"
  exit 1
fi
echo "✅ All Code Comment Standards criteria met"
```
