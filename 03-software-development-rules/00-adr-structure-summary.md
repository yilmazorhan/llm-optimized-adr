# ADR Structure: Optimized for Architectural Decisions Records

## ADR Template Rules:
This document and all ADRs derived from this ADR **MUST** follow RFC 2119 standards for requirement levels:

- **MUST** / **REQUIRED** / **SHALL**: Absolute requirement
- **MUST NOT** / **SHALL NOT**: Absolute prohibition  
- **SHOULD** / **RECOMMENDED**: Strong recommendation with rare exceptions
- **SHOULD NOT** / **NOT RECOMMENDED**: Strong recommendation against with rare exceptions
- **MAY** / **OPTIONAL**: Truly optional, implementation choice
- Every ADR uses the following structure, designed for both human and LLM agent automation. 
- Every text in ADR uses the following structure **MUST** be short and clear.
- Every ADR derived from this template **MUST** be objective, evidence-based and quality metrics and contents **SHOULD NOT** be open to interpretation

## ADR Template

1. **# Issue:** Short issue title
	- Problem statement and context (for developer use to understand requirements)
	- Clearly define problem scope and impact
	- **MUST** be short
2. **# Decision:**
	- Specific solution and implementation details (**MUST** follow exactly)
	- **MUST** Provide concrete, actionable decisions
3. **# Constraints:**
	- Technical/business boundaries (**MUST NOT** violate)
	- **MUST NOT**  be exceeded without explicit architectural review
4. **# Alternatives:**
	- Rejected options with brief reasoning (**MUST** avoid these)
	- **MUST** Document why alternatives were not chosen
	- **MUST** have minimum 1 maximum 2 options.
	- Rejection reasoning **MUST** cite verifiable facts (official documentation, measurable metrics, documented limitations)
	- **MUST NOT** use subjective comparatives (e.g., "superior", "better", "more mature", "best") without measurable evidence
5. **# Rationale:**
	- **MUST** have justification for the decision (use for trade-off logic)
	- **MUST** have provide clear reasoning for the chosen approach
	- Justifications **MUST** reference concrete, verifiable evidence (official docs, metrics, documented constraints)
	- **MUST NOT** contain subjective adjectives or unsubstantiated comparative claims
6. **# Implementation Guidelines:**
	- **MUST** follow Step-by-step instructions (generate code/config as described)
	- **MUST** Follow precisely during implementation
	- **RECOMMENDED** Include practical examples, code snippets, configuration files
	- **RECOMMENDED** Provide exact version numbers and dependency specifications
	- **RECOMMENDED** Include setup scripts, commands, or automation tools
	- May contain troubleshooting guides and validation steps
	- Define naming conventions for all relevant artifacts (classes, methods, files, packages, etc.)
	- Specify coding standards and style requirements where applicable
7. **# Additional Recommendations:**
	- **MUST** Reference relevant documentation, tools, or resources
	- Best practices and related suggestions (apply if relevant)
	- May be applied based on specific implementation context
	- Include optimization techniques and advanced configurations
	
8. **# Success Criteria:**
	- Every compilation and coding process **MUST** successfully pass these criteria.
	- Criteria that **MUST** all be met before considering the ADR successfully implemented.
	- **MUST** Include validation methods and expected results in table format
	- **MUST** Provide validation scripts or commands where applicable