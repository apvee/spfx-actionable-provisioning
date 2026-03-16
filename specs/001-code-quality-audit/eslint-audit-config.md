# ESLint Security Audit Configuration

**Purpose**: Document ESLint rules used for security analysis during the code quality audit.  
**Feature**: 001-code-quality-audit  
**Date**: 2026-03-16

---

## Overview

This document describes the ESLint rules and configuration used for automated security analysis. These rules focus on identifying:

1. Type safety violations
2. Async/await misuse
3. Dangerous function usage
4. Potential injection vectors

---

## Recommended Rules

### Type Safety Rules

```javascript
{
  "@typescript-eslint/no-explicit-any": "error",
  "@typescript-eslint/no-unsafe-assignment": "warn",
  "@typescript-eslint/no-unsafe-call": "warn",
  "@typescript-eslint/no-unsafe-member-access": "warn",
  "@typescript-eslint/no-unsafe-return": "warn",
  "@typescript-eslint/no-unsafe-argument": "warn"
}
```

**Rationale**: Any usage undermines type safety and can mask security issues.

### Async/Await Rules

```javascript
{
  "@typescript-eslint/no-floating-promises": "error",
  "@typescript-eslint/await-thenable": "error",
  "@typescript-eslint/no-misused-promises": "error",
  "@typescript-eslint/promise-function-async": "warn"
}
```

**Rationale**: Unhandled promises can lead to silent failures and security bypasses.

### Dangerous Function Rules

```javascript
{
  "no-eval": "error",
  "no-implied-eval": "error",
  "no-new-func": "error",
  "no-script-url": "error"
}
```

**Rationale**: These functions can execute arbitrary code and are injection vectors.

### Code Quality Rules

```javascript
{
  "no-unused-vars": "off",
  "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
  "no-unreachable": "error",
  "no-constant-condition": "warn",
  "no-debugger": "error"
}
```

---

## Running the Audit

### Option 1: Using existing ESLint config

```bash
npx eslint src/ --ext .ts,.tsx --format json > eslint-results.json
```

### Option 2: Manual rule check

```bash
# Check for explicit any
grep -rn "any" src/ --include="*.ts" --include="*.tsx" | grep -v "node_modules"

# Check for console.log
grep -rn "console\." src/ --include="*.ts" --include="*.tsx"

# Check for eval usage
grep -rn "eval\|Function\(" src/ --include="*.ts" --include="*.tsx"
```

---

## Finding Categories

### Security (Critical/High)

- `no-eval`, `no-implied-eval`, `no-new-func` violations
- Unvalidated user input in API calls
- Parameter substitution without sanitization

### Bug (High/Medium)

- `no-floating-promises` violations
- Unhandled promise rejections
- Missing null checks

### Quality (Medium/Low)

- `no-explicit-any` violations
- Missing type annotations
- Inconsistent error handling

---

## Integration with Audit Report

Findings from ESLint analysis should be:

1. Categorized by severity (Critical/High/Medium/Low)
2. Mapped to Finding IDs (FINDING-XXX)
3. Documented in `audit-report.md` with file location and line numbers
4. Tracked through resolution

---

## Reference

- [TypeScript ESLint Rules](https://typescript-eslint.io/rules/)
- [ESLint Security Plugin](https://github.com/eslint-community/eslint-plugin-security)
- [Project Constitution - Type Safety Principle](../../.specify/memory/constitution.md)
