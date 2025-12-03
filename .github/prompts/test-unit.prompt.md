# Unit Test Coverage Automation Agent

## Context Analysis Phase

### 1. Technology Stack Detection
Analyze project configuration files in this priority:
- **Java/Kotlin**: `pom.xml`, `build.gradle`, `build.gradle.kts`
- **JavaScript/TypeScript**: `package.json`
- **Python**: `requirements.txt`, `pyproject.toml`, `setup.py`
- **.NET**: `*.csproj`, `packages.config`
- **Go**: `go.mod`

Extract from configuration:
- Testing framework (JUnit, TestNG, Jest, Mocha, pytest, NUnit, etc.)
- Assertion libraries
- Mocking frameworks (Mockito, Sinon, unittest.mock, etc.)
- Coverage tools (Jacoco, Istanbul, Coverage.py, etc.)

### 2. Project Pattern Recognition
Scan existing test files in standard test directories:
- Java: `src/test/java/`, `src/test/kotlin/`
- JavaScript/TypeScript: `test/`, `tests/`, `__tests__/`, `*.test.js`, `*.spec.ts`
- Python: `tests/`, `test_*.py`
- .NET: `*.Tests/`, `*.UnitTests/`

Identify and document:
- **Naming Conventions**: Test class/file naming patterns
- **Test Structure**: AAA (Arrange-Act-Assert), Given-When-Then, etc.
- **Common Assertions**: Preferred assertion styles
- **Mocking Patterns**: How dependencies are mocked
- **Setup/Teardown**: @BeforeEach, beforeAll, setUp, fixtures
- **Test Data Builders**: Factory patterns, builders, fixtures
- **Parameterized Tests**: @ParameterizedTest, @DataProvider, pytest.mark.parametrize

### 3. Change Scope Detection
Identify files requiring test coverage:

```bash
# Get modified files in current branch vs origin
git diff origin/main...HEAD --name-only
git diff origin/develop...HEAD --name-only
```

Filter to source files only (exclude):
- Configuration files (*.yml, *.json, *.xml, *.properties)
- Documentation (*.md, docs/)
- Build scripts
- Generated code

Cross-reference with GitHub Copilot `#changes` if provided.

### 4. Coverage Gap Analysis
Parse coverage reports based on detected stack:

**Java (Jacoco)**:
- Location: `target/site/jacoco/index.html`, `build/reports/jacoco/test/html/`
- Parse: `jacoco.xml` for programmatic analysis
- Extract: missed lines, missed branches per class/method

**JavaScript (Istanbul/NYC)**:
- Location: `coverage/lcov-report/index.html`
- Parse: `coverage/coverage-final.json` or `lcov.info`

**Python (Coverage.py)**:
- Location: `htmlcov/index.html`
- Parse: `.coverage` database or `coverage.json`

**Prioritization**:
1. Critical business logic (services, use cases, domain)
2. Modified files in current branch
3. Controllers/handlers with complex logic
4. Utilities with algorithmic complexity
5. Skip: DTOs, simple POJOs, generated code, getters/setters

---

## Implementation Plan Generation

Create a structured checklist with coverage estimation:

```markdown
### Test Implementation Plan - [Branch Name] - [Date]
**Target**: 90%+ line coverage, 85%+ branch coverage

#### Phase 1: [FileName1.java] - Critical Business Logic
Current Coverage: 45% | Target: 90%+

- [ ] **Method**: `calculateDiscount(Order order)` 
  - Lines: 45-67 (23 uncovered)
  - Branches: 4/8 covered
  - Tests needed: happy path, null order, zero items, discount tiers
  - **Est. coverage gain**: +12%

- [ ] **Method**: `validatePayment(Payment payment)`
  - Lines: 89-103 (15 uncovered)
  - Branches: 2/6 covered
  - Tests needed: valid payment, invalid card, expired card, insufficient funds
  - **Est. coverage gain**: +8%

**Phase 1 Target**: 65% → 90%

#### Phase 2: [FileName2.java] - Service Layer
Current Coverage: 60% | Target: 90%+

- [ ] **Method**: `processOrder(OrderRequest request)`
  - Lines: 120-145
  - Tests needed: success flow, validation errors, external API failure
  - **Est. coverage gain**: +15%

**Phase 2 Target**: 60% → 90%

---
**Overall Project Coverage**: 58% → 90%+ (Target)
```

---

## Execution Protocol

Execute **iteratively** for each unchecked task:

### Step 1: Generate Test Implementation
- Follow **exact project patterns** identified in Phase 2
- Use **same naming conventions** as existing tests
- Apply **same mocking approach** found in project
- Structure tests with **same format** (AAA, Given-When-Then, etc.)

**DO NOT**:
- Invent new test patterns
- Hallucinate test data not seen in existing tests
- Create overly complex mocks
- Test simple getters/setters without logic

**DO**:
- Use realistic test data from existing tests
- Cover: happy path, edge cases, error scenarios
- Test boundary conditions
- Verify exception handling

### Step 2: Execute Tests
Run test command based on stack:
```bash
# Java Maven
mvn clean test

# Java Gradle
./gradlew test

# JavaScript/TypeScript
npm test
# or
npm run test:unit

# Python
pytest
# or
python -m pytest

# .NET
dotnet test
```

Verify: All tests pass ✅

### Step 3: Generate Coverage Report
```bash
# Java Maven
mvn jacoco:report

# Java Gradle
./gradlew jacocoTestReport

# JavaScript
npm run test:coverage
# or
jest --coverage

# Python
pytest --cov=src --cov-report=html --cov-report=term

# .NET
dotnet test /p:CollectCoverage=true /p:CoverageReportFormat=cobertura
```

### Step 4: Analyze Coverage for Current File
Parse coverage report **specifically for the file being tested**:
- Extract: current line coverage %, branch coverage %
- Identify: remaining uncovered lines/branches

### Step 5: Decision Point
**IF** coverage >= 90% for current file:
- ✅ Mark task as complete
- 📝 Log: "✅ [FileName] - Coverage: X% (Target achieved)"
- ➡️ Move to next unchecked task

**ELSE IF** coverage < 90%:
- 🔍 Analyze: Which lines/branches still uncovered?
- 📝 Generate: Supplementary tests for gaps
- 🔄 Return to Step 2 (run tests again)
- ⚠️ Max 3 iterations per file - if still not 90%, document blockers

### Step 6: Update Checklist
Update the Implementation Plan with:
- Current coverage percentage
- Completed tasks marked with ✅
- Timestamp of completion

### Step 7: Verify Overall Target
After each file completion:
- Check: overall project coverage
- IF overall >= 90%: **STOP** - objective achieved
- ELSE: continue to next unchecked task

---

## Constraints & Quality Gates

### Mandatory Rules
1. ✅ **Evidence-Based Only**: Every test must follow existing project patterns
2. ✅ **Verify Execution**: Tests must actually run and pass before marking complete
3. ✅ **No Hallucination**: Do not invent frameworks, annotations, or patterns not in project
4. ✅ **Incremental Progress**: One file at a time, verify before proceeding
5. ✅ **Coverage Validation**: Parse actual coverage reports, not estimates

### Exclusions
Do NOT create tests for:
- Simple DTOs/POJOs without logic
- Auto-generated code (Lombok @Data, MapStruct mappers, etc.)
- Configuration classes with only @Bean/@Component
- Simple getters/setters
- Third-party library code

### Stop Conditions
Stop execution when:
- ✅ All modified files reach 90%+ coverage
- ✅ Overall project coverage >= 90%
- ⚠️ 3 iterations on same file without reaching target (document why)
- 🛑 Test failures unrelated to new tests (fix first)

---

## Output Format

### Final Report Structure

```markdown
## Unit Test Coverage Automation Report
**Date**: [timestamp]
**Branch**: [branch-name]
**Initial Coverage**: X%
**Final Coverage**: Y%
**Target Achievement**: ✅ / ⚠️

---

### 1. Technology Stack Summary
- **Language**: [detected]
- **Framework**: [detected]
- **Test Framework**: [detected]
- **Mocking**: [detected]
- **Coverage Tool**: [detected]

---

### 2. Detected Test Patterns
- **Naming**: [pattern]
- **Structure**: [AAA, Given-When-Then, etc.]
- **Mocking Style**: [when/given, @Mock, mock(), etc.]
- **Assertions**: [assertThat, expect, assert, etc.]

---

### 3. Coverage Gap Analysis (Pre-Implementation)
| File | Initial Coverage | Uncovered Lines | Priority |
|------|------------------|-----------------|----------|
| FileA.java | 45% | 67 | High |
| FileB.java | 60% | 34 | Medium |

---

### 4. Implementation Execution Log

#### Phase 1: FileA.java
- ✅ Test: `shouldCalculateDiscountForValidOrder()` → Coverage: 45% → 58%
- ✅ Test: `shouldThrowExceptionForNullOrder()` → Coverage: 58% → 65%
- ✅ Test: `shouldApplyTieredDiscounts()` → Coverage: 65% → 78%
- ✅ Test: `shouldHandleZeroItemsOrder()` → Coverage: 78% → 92%
- **Result**: ✅ 92% coverage achieved

#### Phase 2: FileB.java
- ✅ Test: `shouldProcessOrderSuccessfully()` → Coverage: 60% → 75%
- ✅ Test: `shouldHandleExternalAPIFailure()` → Coverage: 75% → 91%
- **Result**: ✅ 91% coverage achieved

---

### 5. Final Coverage Report
| File | Initial | Final | Status |
|------|---------|-------|--------|
| FileA.java | 45% | 92% | ✅ |
| FileB.java | 60% | 91% | ✅ |
| **Overall** | **58%** | **91%** | ✅ |

---

### 6. Blockers & Recommendations
- ⚠️ [File X]: Could not reach 90% due to [reason] - recommend [action]
- 💡 Consider integration tests for [scenario]
```

---

## Usage Instructions

### In GitHub Copilot Chat:
1. Open the file(s) you want to test
2. Reference this prompt: `@workspace /new #file:unit-test-coverage-automation.md`
3. Or copy this prompt into Copilot Chat
4. Let the agent execute all phases automatically
5. Review the execution log and validate tests manually if needed

### Command Line Integration:
This prompt can be used with GitHub Copilot CLI or workspace commands to automate the full cycle from analysis to achieving coverage targets.

---

**Version**: 1.0
**Last Updated**: December 2025
**Compatible with**: GitHub Copilot Chat, GitHub Copilot Workspace
