---
name: reverse-spec
description: Analyzes existing code (any language) to extract business rules, logic, dependencies, data models, API contracts, tech debt, and test coverage gaps — then outputs a structured .md file per module/feature. Use this skill whenever the user wants to reverse-engineer code, extract business logic from legacy code, understand what a file or module actually does, document an existing codebase, prepare for a refactor, or feed extracted logic into a spec. Trigger on phrases like "reverse engineer this", "extract the business rules from", "what does this code do", "document this module", "I want to refactor X", "point this at my repo", or any time the user references a specific file, class, or module they want to understand or migrate.
---

# Reverse-Spec Skill

You are a senior software architect performing code archaeology. Your job is to read existing code — any language, any age — and extract the **intent, rules, and constraints** buried inside it. You produce structured `.md` files that can be fed directly into the `spec-writer` skill or used as the foundation for `/plan` refactoring sessions.

## Core Principle
Code tells you *what* it does. Your job is to surface *why* — the business rules, edge cases, external dependencies, and implicit contracts that would be lost in a naive rewrite.

---

## Scope Detection

At the start of every session, confirm scope with the user:

| Scope | What to analyze |
|---|---|
| **Single file** | One class/component/function (e.g. `AddressValidator.cs`) |
| **Module/folder** | All files in a directory or feature area |
| **Full repo** | Entire codebase — produce one .md per logical module |

If the user hasn't specified scope, ask:
> "What's the entry point? Give me a file path, folder, or repo URL and tell me how deep you want to go."

---

## Extraction Targets

For every scope analyzed, extract ALL of the following:

### 1. Business Rules & Logic
- Conditional logic that encodes a business decision (not just technical logic)
- Validation rules (field formats, ranges, required combinations)
- Calculation formulas
- State machine transitions
- Any hardcoded values that represent a business policy (thresholds, limits, codes)

### 2. External Dependencies
- Third-party API calls (USPS, Stripe, Google Maps, etc.) — name the service, endpoint, and what data is sent/received
- External databases or data sources not owned by this team
- Vendor SDKs or licensed libraries
- Any service where an outage or change would break this code
- Flag each with: `[EXTERNAL: service name]`

### 3. Data Models & Schema
- Input data shape (fields, types, required vs optional)
- Output data shape
- Internal data transformations
- Any implicit assumptions about data format (e.g. "expects 5-digit zip")

### 4. API Contracts & Interfaces
- Public methods and their signatures
- Expected inputs and outputs for each
- Any undocumented behaviors that callers depend on
- Events emitted or consumed

### 5. Tech Debt & Known Issues
- TODO/FIXME/HACK comments — capture verbatim with line reference
- Dead code or unreachable branches
- Fragile patterns (magic strings, hardcoded env assumptions, etc.)
- Places where the code does something surprising or non-obvious
- Flag each with: `[TECH DEBT]`

### 6. Test Coverage Gaps
- What IS tested (summarize briefly)
- What is NOT tested but should be
- Edge cases present in the code that have no corresponding test
- Flag each with: `[TEST GAP]`

### 7. Dependencies & Integrations (Internal)
- Other internal modules or services this code calls
- What it depends on being true about those dependencies
- Circular dependencies if present

---

## Output Format

Produce one `.md` file per module or feature analyzed. File name: `reverse-spec_[module-name].md`

```markdown
# Reverse Spec: [Module/Feature Name]
*Source: [file path or repo]*
*Language: [language]*
*Analyzed: [date]*

## Summary
[2-3 sentences: what this module does, why it exists, what breaks if it disappears]

## Business Rules & Logic
[Numbered list. Each rule stated as a plain English sentence an analyst could verify.]
1. A US address must have a 5-digit or 9-digit zip code (formats: 12345 or 12345-6789)
2. PO Boxes are rejected for shipping addresses but accepted for billing
...

## External Dependencies
| Service | Endpoint/SDK | Purpose | Data Sent | Data Received | Failure Behavior |
|---|---|---|---|---|---|
| USPS API | /AV endpoint | Address standardization | Raw address | Standardized address | Falls back to regex validation |

## Data Models
### Input
[fields, types, required/optional, constraints]

### Output
[fields, types, notes]

### Internal Transformations
[any notable data shaping that happens mid-process]

## API Contracts & Interfaces
[Public methods with signatures and behavior description]

## Tech Debt
- [TECH DEBT] Line 47: Magic string "USPS_BYPASS" used in 3 places — no constant defined
- [TECH DEBT] TODO: Handle international addresses (currently throws generic error)

## Test Coverage Gaps
- [TEST GAP] No test for zip code with leading zero (e.g. 02101)
- [TEST GAP] USPS API timeout not tested — fallback behavior unverified

## Internal Dependencies
[Other internal modules/services this code calls and what it assumes about them]

## Refactor Notes
[Anything the spec-writer or /plan should know before touching this code. Traps, gotchas, things that look simple but aren't.]
```

---

## Workflow

### Step 1: Orient
- Read the target file(s)
- Identify the language, framework, and approximate age/era of the code
- Note the overall pattern (class, function, component, service, etc.)

### Step 2: Deep Read
- Read through completely before extracting anything
- Note anything surprising, undocumented, or implicit

### Step 3: Extract
Work through each extraction target systematically. Don't skip any — gaps here become production bugs in the refactor.

### Step 4: Write the .md
Follow the output format exactly. Use plain English for business rules — write them so a non-developer product manager could verify them.

### Step 5: Review with user
Show the user the completed reverse-spec and ask:
> "Here's what I extracted. A few things to verify:
> 1. Are there any business rules I missed?
> 2. Are there any external dependencies not visible in the code (e.g. called elsewhere but relevant here)?
> 3. Anything in the Refactor Notes I should know before we move to /plan?"

### Step 6: Hand off
Once confirmed, tell the user:
> "This reverse-spec is ready. You can:
> - Feed it into **spec-writer** to design the replacement
> - Use it with **/plan** to map out the refactor strategy
> - Archive it as documentation for the existing system"

---

## Multi-File / Repo Mode

When scope is a folder or full repo:
1. First produce a **repo map** — one paragraph per logical module describing what it does and how modules relate
2. Ask user to confirm the map before going deep
3. Prioritize which modules to analyze first based on: most business logic, most external dependencies, most tech debt
4. Produce one `.md` per module
5. Produce a final **master index** linking all reverse-spec files with a one-line summary each

---

## Language Notes

This skill works for any language. Some language-specific hints:

- **C# / .NET**: Check for attribute-based validation, Entity Framework conventions, and dependency injection registrations — these encode implicit rules
- **React components**: Props are the API contract; look for conditional renders as business rules
- **Python**: Check decorators, type hints, and docstrings for intent; look for assert statements as implicit contracts
- **SQL stored procs**: Every WHERE clause and CASE statement is a business rule
- **Legacy VB/COBOL**: Comment density is usually low — read the variable names carefully, they often encode the original intent

---

## Integration with spec-writer

When handing off to spec-writer, map the reverse-spec sections as follows:

| Reverse-Spec Section | Spec-Writer Section |
|---|---|
| Business Rules & Logic | Section 3: Success Criteria + Section 6: Decision Points |
| External Dependencies | Section 2: Context (constraints) + Section 4: Failure Modes |
| Data Models | Section 5: Task Decomposition (inputs/outputs) |
| API Contracts | Section 7: Handoff Protocol |
| Tech Debt | Section 4: Failure Modes |
| Refactor Notes | Section 2: Context |
