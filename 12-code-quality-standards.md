# Issue: Code Quality Standards for Cloud-Native Applications

Standardized code quality practices required to ensure consistency, maintainability, and collaboration effectiveness across cloud-native microservices. Without unified standards, teams experience reduced productivity and difficult code reviews.

# Decision:

Adopt Google Java Style Guide with Quarkus-specific modifications, enforced through automated tooling integrated into the Maven build pipeline.

- Spotless **MUST** be configured for automated code formatting enforcement.
- Checkstyle **MUST** be configured to validate Google Java Style compliance.
- PMD **MUST** be configured with a custom ruleset for complexity and code quality rules.
- SpotBugs with FindSecBugs **MUST** be configured for bug detection and security analysis.
- All quality plugins **MUST** run automatically during the build lifecycle and **MUST** fail the build on any violation.
- Tests **MUST NOT** be skipped in standard `build` and `verify` commands.

# Constraints:

- All code patterns **MUST** support GraalVM native image compilation.
- **MUST** be compatible with CDI, reactive programming, and Quarkus framework patterns.
- **MUST** provide consistent formatting across IntelliJ IDEA, VS Code, and Eclipse.
- **MUST NOT** introduce breaking changes to existing functionality during implementation.
- Tests **MUST NOT** be skipped in standard build and verify commands; `-DskipTests` **MUST NOT** appear in `./mvnw clean verify` invocations.
- Quality plugins **MUST** be bound to the default Maven lifecycle phases (`validate`, `verify`).

# Alternatives:

1. **Manual Code Reviews Only (no automated formatting/linting)**: Rely on human reviewers to catch formatting and style issues during pull requests. Rejected because: Google's engineering practices documentation states that automated formatting eliminates formatting as a review topic entirely (https://google.github.io/styleguide/javaguide.html). Without automation, each reviewer applies their own interpretation of style rules, and studies on code review effectiveness show formatting comments account for 10-15% of review comments in projects without auto-formatters (https://www.microsoft.com/en-us/research/publication/code-reviewing-in-the-trenches-understanding-challenges-and-best-practices/).

2. **Oracle Java Code Conventions**: The original Sun/Oracle Java code conventions document. Rejected because: The document was last updated in April 1999 (https://www.oracle.com/java/technologies/javase/codeconventions-introduction.html) and does not cover Java features introduced since Java 5: generics, annotations, lambda expressions, records, sealed classes, pattern matching, or text blocks. Google Java Style Guide is actively maintained and covers through Java 21 features.

# Rationale:

Google Java Style Guide provides machine-enforceable formatting rules via the `google-java-format` tool, which Spotless integrates directly (https://github.com/diffplug/spotless/tree/main/plugin-maven#google-java-format). Checkstyle provides the official Google Checks configuration (`/google_checks.xml`) bundled with the Checkstyle distribution (https://checkstyle.sourceforge.io/google_style.html). PMD's complexity rules (CyclomaticComplexity, CognitiveComplexity) provide measurable thresholds that prevent methods from exceeding defined limits — these are not subjective review opinions but build-failing numeric checks (https://pmd.github.io/latest/pmd_rules_java_design.html). SpotBugs with FindSecBugs detects security vulnerabilities mapped to CWE identifiers at compile time (https://find-sec-bugs.github.io/). Running all four tools in the Maven lifecycle ensures every commit meets quality standards before merge.

# Implementation Guidelines:

## Maven Plugin Configuration:
Spotless, Checkstyle, PMD, and SpotBugs are **MUST** in the default Maven build lifecycle. All code quality checks run automatically during the `validate` phase (Spotless, Checkstyle, PMD) and `verify` phase (SpotBugs). The build **fails** if any violations are detected.

## PMD Custom Ruleset:
A custom PMD ruleset is configured at `build/pmd-ruleset.xml` with the following mandatory rules:

| Rule | Description | Threshold |
|------|-------------|-----------|
| `AvoidBooleanMethodParameters` | Avoid boolean parameters (encourages method overloading) | Enabled |
| `AvoidDeeplyNestedIfStmts` | Prevents deeply nested if statements | Max depth: 3 |
| `CognitiveComplexity` | Limits cognitive complexity of methods | Max: 15 |
| `CyclomaticComplexity` | Limits cyclomatic complexity | Method: 10, Class: 80 |
| `ExcessiveClassLength` | Prevents overly long classes | Max: 1000 lines |
| `ExcessiveParameterList` | Limits method parameters | Max: 7 parameters |
| `ExcessivePublicCount` | Limits public members in a class | Max: 45 |
| `NcssConstructorCount` | Limits constructor NCSS | Max: 60 |
| `NcssCount` | Limits Non-Commenting Source Statements | Method: 60, Class: 1500 |
| `NcssMethodCount` | Limits method NCSS | Max: 60 |
| `NcssTypeCount` | Limits type/class NCSS | Max: 1500 |
| `StdCyclomaticComplexity` | Standard cyclomatic complexity | Max: 10 |
| `TooManyFields` | Limits number of fields in a class | Max: 15 |
| `UnusedMethod` | Detects unused private methods | Enabled |

## SpotBugs Configuration:
SpotBugs runs with maximum effort and medium threshold, including FindSecBugs plugin for security analysis. Builds fail on any detected bugs.

## Maven Verification Commands:

**Apply and verify Spotless formatting:**
```bash
./mvnw spotless:apply
./mvnw spotless:check
```

**Verify Checkstyle Plugin:**
```bash
./mvnw spotless:apply
./mvnw spotless:check
./mvnw checkstyle:check
```

**Verify PMD and SpotBugs:**
```bash
./mvnw spotless:apply
./mvnw spotless:check
./mvnw pmd:check
./mvnw spotbugs:check
```

**All style and quality checks:**
```bash
./mvnw spotless:apply
./mvnw spotless:check
./mvnw verify
```

# Additional Recommendations:

- **RECOMMENDED** Use git pre-commit hooks (e.g., via `pre-commit` framework) to run `./mvnw spotless:apply` before commits.
- **RECOMMENDED** Review PRs modifying build commands for test skip violations (`-DskipTests`, `-Dmaven.test.skip`).
- **RECOMMENDED** Generate PMD and SpotBugs HTML reports for developer feedback: `./mvnw pmd:pmd spotbugs:spotbugs`.
- **RECOMMENDED** Configure IDE formatter settings to match `google-java-format` for real-time formatting.
- **MUST** Reference ADR-08 (Quarkus Framework), ADR-10 (Hexagonal Architecture), ADR-18 (Security OWASP).
- See Google Java Style Guide: https://google.github.io/styleguide/javaguide.html
- See Spotless Maven Plugin: https://github.com/diffplug/spotless/tree/main/plugin-maven
- See Checkstyle Google Checks: https://checkstyle.sourceforge.io/google_style.html
- See PMD Java Rules: https://pmd.github.io/latest/pmd_rules_java.html
- See SpotBugs: https://spotbugs.github.io/
- See FindSecBugs: https://find-sec-bugs.github.io/

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Spotless Configured | `grep spotless pom.xml` | Plugin present |
| Checkstyle Configured | `grep checkstyle pom.xml` | Plugin present |
| PMD Configured | `grep pmd pom.xml` | Plugin present |
| SpotBugs Configured | `grep spotbugs pom.xml` | Plugin present |
| Spotless Check Passes | `./mvnw spotless:check` | Exit code 0 |
| Checkstyle Passes | `./mvnw checkstyle:check` | Exit code 0 |
| PMD Passes | `./mvnw pmd:check` | Exit code 0 |
| SpotBugs Passes | `./mvnw spotbugs:check` | Exit code 0 |
| No Formatting Violations | `./mvnw spotless:apply` | No files changed |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Code Quality Standards..."

# Check for quality plugins in pom.xml
PLUGINS=("spotless" "checkstyle" "pmd" "spotbugs")
for plugin in "${PLUGINS[@]}"; do
  if grep -rq "$plugin" pom.xml */pom.xml 2>/dev/null; then
    echo "✅ $plugin plugin configured"
  else
    echo "❌ $plugin plugin not found"
    exit 1
  fi
done

# Run Spotless apply + check
./mvnw spotless:apply -q
echo "✅ Spotless apply completed"

./mvnw spotless:check -q
echo "✅ Spotless check passed"

# Run Checkstyle
./mvnw checkstyle:check -q
echo "✅ Checkstyle passed"

# Run PMD
./mvnw pmd:check -q
echo "✅ PMD check passed"

# Run SpotBugs
./mvnw spotbugs:check -q
echo "✅ SpotBugs check passed"

# Full verify (MUST NOT skip tests per ADR-12 constraints)
./mvnw clean verify -q
echo "✅ Full build verification successful (tests included)"

echo "✅ All Code Quality Standards criteria met"
```
