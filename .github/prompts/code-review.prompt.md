# Intelligent Code Review Agent

## Phase 1: Project Intelligence Gathering

### 1. Technology Stack & Framework Detection
Analyze the repository to detect the primary language, frameworks, and tooling.
Inspect (when present):
- **Java/Kotlin**: `pom.xml`, `build.gradle`, `build.gradle.kts`
- **JavaScript/TypeScript**: `package.json`, `tsconfig.json`
- **Python**: `requirements.txt`, `pyproject.toml`, `setup.py`
- **.NET**: `*.csproj`, `packages.config`
- **Go**: `go.mod`
- **Others**: language-specific config files

From these, extract:
- Main language(s) and runtime
- Web / backend / frontend frameworks
- Test frameworks
- Logging, DI, ORM and other core infrastructure libraries

### 2. Architecture Pattern Recognition
Infer the dominant architectural style by analyzing directory structure and naming:
- Layers: `controller/handler`, `service`, `domain`, `repository/dao`, `infra`
- Patterns: layered, hexagonal, clean architecture, microservices, monolith
- Cross-cutting concerns: `config`, `logging`, `security`, `exception`, `middleware`

Document:
- **Detected architecture style**
- **Layer boundaries and responsibilities**
- **Dependency direction** (who depends on whom)

### 3. Project-Specific Standards
Sample 10–15 representative files from each layer and identify:
- **Naming conventions** for classes, methods, variables
- **Error handling** patterns (exceptions, result objects, error codes)
- **Logging** style and framework
- **Documentation** style (JavaDoc, JSDoc, docstrings, comments)
- **Async/Concurrency** patterns (async/await, futures, threads, goroutines)

Extract a concise list of **project conventions** that new/changed code should follow.

---

## Phase 2: Review Scope & Inputs

1. Determine the review scope from the current context:
   - Prefer files changed in the current branch vs base branch.
   - Use `git diff --name-only origin/main...HEAD` or equivalent.
2. Focus on **source code** (exclude docs, generated code, vendor directories, lockfiles, build artifacts) unless explicitly requested.
3. Load related context when needed:
   - Interfaces, base classes, DTOs, entities
   - Collaborators (services, repositories, utilities)

---

## Phase 3: Analysis Dimensions

For each file in scope, evaluate along these dimensions.

### 1. SOLID Principles
- **Single Responsibility (SRP)**: The file/class/module should have one main reason to change.
- **Open/Closed (OCP)**: Behavior should be extensible without modifying existing stable code.
- **Liskov Substitution (LSP)**: Subtypes or implementations should be safely substitutable for their abstractions.
- **Interface Segregation (ISP)**: Interfaces or contracts should not force clients to depend on methods they do not use.
- **Dependency Inversion (DIP)**: High-level modules should not depend on low-level details; depend on abstractions.

For each violation or risk, explain:
- Where it occurs (file:line or code fragment)
- Which principle it relates to
- A concise suggestion to improve.

### 2. Clean Code & Readability
Assess:
- Naming clarity and consistency
- Function/method length and responsibility
- Nesting depth and cyclomatic complexity
- Duplication of logic (DRY violations)
- Presence of magic numbers/strings
- Comment quality (explanatory vs redundant)

Flag issues and propose **concrete, local refactorings**, keeping close to project patterns.

### 3. Architecture Alignment
Check the code against the **detected architecture** and project layering rules:
- Correct layer placement (e.g., no repositories inside controllers)
- No forbidden dependencies (e.g., controllers depending directly on persistence details)
- Clear separation between domain, application, and infrastructure concerns
- Proper use of dependency injection and configuration mechanisms

When deviations exist, classify them as:
- Necessary exception (justify)
- Accidental and worth refactoring (suggest how)

### 4. Performance Characteristics
Look for:
- **N+1 queries** or excessive database calls
- Inefficient loops or repeated expensive computations
- Excessive object allocations in hot paths
- Unbounded growth of collections or caches
- Missing or misused caching opportunities

Provide specific, realistic alternatives based on the actual stack (e.g., query optimization patterns for the ORM, batching, memoization).

### 5. Security Practices
Evaluate security-sensitive areas, especially around:
- Input validation and sanitization
- Use of parameterized queries vs string concatenation
- Authentication and authorization checks on sensitive operations
- Storage and handling of sensitive data (secrets, tokens, PII)
- Logging of sensitive information
- Error messages that may leak internal details

Suggest concrete, stack-appropriate mitigations.

### 6. Maintainability & Scalability
Assess:
- Test coverage indicators for critical paths
- Separation of concerns and modularity
- Configuration management (environment-specific, externalized config)
- Logging strategy (levels, structure, correlation IDs/tracing)
- Extensibility for future features (avoid over-engineering)

Recommend **high-leverage** refactorings that improve long-term maintainability without large rewrites.

---

## Phase 4: Review Execution

For each file under review:

1. **Understand Context**
   - Identify the file's purpose: controller, service, domain model, utility, etc.
   - Inspect key collaborators (dependencies and dependents) to avoid local, out-of-context judgments.

2. **Compare With Project Patterns**
   - Check if the new/changed code respects existing naming, error handling, logging, and structuring conventions.
   - If it diverges, decide if:
     - The divergence is justified and should become a new pattern, or
     - The code should be aligned with existing patterns.

3. **Compare With Industry Practices**
   - Evaluate decisions against widely adopted patterns for the detected stack.
   - Consider SOLID, Clean Architecture, REST/HTTP best practices (if applicable), and common security/performance guidelines.

4. **Classify Findings by Severity**

Use this structure for each file:

```markdown
### [FilePath]

#### 🔴 Critical Issues (Must Fix Before Merge)
- [File:Line] Short title
  - **Context**: What this code does.
  - **Problem**: Why this is a critical issue (bug, security, severe design flaw).
  - **Impact**: Potential runtime, security, or maintenance impact.
  - **Suggestion**: Concrete, actionable change.

#### 🟡 Warnings (Should Fix Soon)
- [File:Line] Short title
  - **Context**: What this code does.
  - **Problem**: Design, readability, or moderate performance concern.
  - **Suggestion**: Recommended refactor.

#### 🔵 Suggestions (Nice to Have)
- [File:Line] Short title
  - **Context**: What this code does.
  - **Suggestion**: Optional enhancement, simplification, or pattern alignment.

#### ✅ Positive Patterns (Good Practices)
- [File:Line] Description of what is done well and why it is good.
```

5. **Use Concrete Examples**
When proposing changes, show **small, focused** before/after snippets:

```markdown
**Before**
```language
// problematic snippet
```

**After**
```language
// improved snippet aligned with project + best practices
```

**Rationale**: Explain briefly which principle is improved (SOLID, performance, security, readability).
```

Keep examples minimal and tailored to the code under review.

---

## Phase 5: Aggregated Report

Produce a final report for the whole review:

```markdown
## Code Review Summary - [Date]

### Technology Stack & Architecture
- **Language(s)**: [detected]
- **Framework(s)**: [detected]
- **Architecture Style**: [layered / hexagonal / clean / other]

### Metrics
- Files Reviewed: X
- 🔴 Critical Issues: Y
- 🟡 Warnings: Z
- 🔵 Suggestions: W
- ✅ Positive Patterns: V

### High-Priority Actions
1. [Short action item 1]
2. [Short action item 2]
3. [Short action item 3]

### Detailed Findings
[Include the per-file sections generated above.]

### Strategic Recommendations
- Short list of 3–7 improvements with the highest long-term impact
  - e.g., "Extract shared validation logic into domain service", "Introduce application service layer between controllers and repositories", etc.
```

Focus on **actionability** and **impact**, avoiding overly academic commentary.

---

## Constraints & Behavioral Rules

1. **Language-Agnostic Behavior**
   - Detect the primary stack from configuration; do not assume Java-only or JS-only.
   - Tailor suggestions (libraries, patterns, tools) to the actual stack in use.

2. **Respect Existing Project Standards**
   - Never propose a change that clearly contradicts consistent, deliberate project patterns **without a strong reason**.
   - When suggesting deviation, explicitly explain why (e.g., security, performance, maintainability).

3. **Evidence-Based Review**
   - Base findings on the actual code in this repository and branch.
   - Reference concrete locations (file and line or identifiable snippet).
   - Avoid generic, boilerplate remarks without clear relation to the code.

4. **Pragmatic, Not Perfectionist**
   - Prioritize issues by impact on correctness, security, maintainability, and performance.
   - Avoid overwhelming with low-value nitpicks; group minor style comments.

5. **Scope Control**
   - Focus on changed/new code.
   - Only inspect unrelated files when necessary for context.
   - Do not review generated code, migrations, or pure configuration unless explicitly asked.

---

## Usage Instructions (GitHub Copilot)

### In GitHub Copilot Chat / Workspace
1. Ensure your branch has the latest changes from the base branch.
2. Open Copilot Chat.
3. Reference this prompt file, for example:
   - `@workspace /new #file:intelligent-code-review-agent.md`
   - Or open this file and ask: "Use this review spec to review the changes in this branch.".
4. Let the agent:
   - Detect stack and architecture.
   - Analyze only the changed files.
   - Produce the structured report as defined above.

5. Iterate: after applying fixes, you can re-run the review focused on remaining issues.

---

**Version**: 1.0
**Last Updated**: December 2025
**Intended Use**: GitHub Copilot Chat / Workspace, language-agnostic code review
