# Data Model: Code Quality Audit

**Feature**: 001-code-quality-audit  
**Date**: 2026-03-16  
**Input**: [research.md](./research.md)

## Entity Definitions

### AuditFinding

Represents a single issue identified during the code quality audit.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✓ | Unique identifier (format: `FINDING-XXX`) |
| `title` | string | ✓ | Brief descriptive title |
| `severity` | enum | ✓ | `Critical` \| `High` \| `Medium` \| `Low` |
| `category` | enum | ✓ | `Security` \| `Bug` \| `Quality` \| `Performance` |
| `location` | Location | ✓ | File path and line number |
| `status` | enum | ✓ | `Open` \| `InProgress` \| `Fixed` \| `Deferred` |
| `description` | string | ✓ | Detailed explanation of the issue |
| `codeExample` | CodeExample | | Before/after code snippets |
| `resolution` | string | | Description of applied fix |
| `relatedFindings` | string[] | | IDs of related findings |
| `detectedBy` | enum | ✓ | `Automated` \| `Manual` |
| `detectedDate` | date | ✓ | When the finding was identified |
| `fixedDate` | date | | When the fix was applied |

### Location

Specifies where in the codebase an issue was found.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | string | ✓ | Relative path from project root |
| `line` | number | | Line number (1-indexed) |
| `endLine` | number | | End line for multi-line issues |
| `function` | string | | Function or method name |
| `module` | enum | ✓ | `core` \| `provisioning` \| `provisioning-ui` |

### CodeExample

Shows before and after code for clarity.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `before` | string | ✓ | Original problematic code |
| `after` | string | | Fixed code (if applicable) |
| `language` | string | | Code language (default: `typescript`) |

---

## Severity Definitions

### Critical (P0)

- **Impact**: Security breach, data exposure, unauthorized access
- **Response Time**: Immediate
- **Examples**: 
  - Credential leakage
  - Injection vulnerabilities
  - Unvalidated permissions

### High (P1)

- **Impact**: Incorrect behavior, data corruption, crashes
- **Response Time**: Within feature scope
- **Examples**:
  - Unhandled exceptions
  - Race conditions
  - Missing null checks

### Medium (P2)

- **Impact**: Maintainability, potential future bugs
- **Response Time**: Within feature scope
- **Examples**:
  - Implicit `any` types
  - Missing documentation
  - Complex functions

### Low (P3)

- **Impact**: Code clarity, minor improvements
- **Response Time**: Within feature scope
- **Examples**:
  - Naming inconsistencies
  - Minor style issues
  - Redundant code

---

## Category Definitions

### Security

Issues that could lead to:
- Unauthorized data access
- Credential exposure
- Injection attacks
- Permission bypass

### Bug

Issues that cause:
- Incorrect program behavior
- Crashes or exceptions
- Silent failures
- Data corruption

### Quality

Issues affecting:
- Type safety
- Code readability
- Maintainability
- Documentation coverage

### Performance

Issues impacting:
- Execution speed
- Memory usage
- API call efficiency
- Resource cleanup

---

## Audit Report Structure

The `audit-report.md` file will follow this structure:

```markdown
# Code Quality Audit Report

**Generated**: YYYY-MM-DD
**Auditor**: GitHub Copilot
**Scope**: src/core/, src/provisioning/, src/provisioning-ui/

## Executive Summary

| Severity | Count | Fixed | Remaining |
|----------|-------|-------|-----------|
| Critical | X | X | 0 |
| High | X | X | 0 |
| Medium | X | X | 0 |
| Low | X | X | 0 |

## Findings by Module

### src/core/

#### [FINDING-001] Title
...

### src/provisioning/

#### [FINDING-XXX] Title
...

### src/provisioning-ui/

#### [FINDING-XXX] Title
...

## Verification Checklist

- [ ] All Critical findings fixed
- [ ] All High findings fixed
- [ ] All Medium findings fixed
- [ ] All Low findings fixed
- [ ] Build passes
- [ ] Type check passes
- [ ] No regressions observed
```

---

## State Transitions

```
┌──────────┐     ┌─────────────┐     ┌───────┐
│   Open   │────▶│ InProgress  │────▶│ Fixed │
└──────────┘     └─────────────┘     └───────┘
      │                                   ▲
      │          ┌──────────┐             │
      └─────────▶│ Deferred │─────────────┘
                 └──────────┘
                 (with justification)
```

**Open**: Finding identified, not yet addressed  
**InProgress**: Fix being developed  
**Fixed**: Fix applied and verified  
**Deferred**: Intentionally postponed (requires justification)

---

## Relationship Model

```
AuditFinding
    │
    ├── Location (1:1)
    │       └── module: core | provisioning | provisioning-ui
    │
    ├── CodeExample (0:1)
    │       └── before/after code
    │
    └── relatedFindings (0:N)
            └── other AuditFinding IDs
```

---

## Validation Rules

1. **ID Uniqueness**: Each finding must have a unique `FINDING-XXX` ID
2. **Location Required**: Every finding must specify file and module
3. **Critical Resolution**: Critical findings cannot be deferred
4. **Deferred Justification**: Deferred status requires documented rationale
5. **Fix Verification**: Fixed status requires resolution description

---

## Metrics Tracking

| Metric | Definition |
|--------|------------|
| Total Findings | Count of all identified issues |
| Fix Rate | (Fixed / Total) × 100% |
| Critical Resolution Time | Time from detection to fix for Critical items |
| Module Distribution | Findings per module (core/provisioning/ui) |
| Category Distribution | Findings per category (Security/Bug/Quality/Performance) |
| Detection Method | Automated vs Manual detection ratio |

---

## Success Criteria Mapping

| Success Criteria | Data Model Support |
|------------------|-------------------|
| SC-001: Zero findings remain | `status` field tracking |
| SC-002: 100% bugs resolved | `category='Bug'` + `status='Fixed'` |
| SC-003: TypeScript strict | `category='Quality'` findings |
| SC-004: JSDoc coverage | `category='Quality'` + documentation findings |
| SC-005: No regressions | Verification checklist |
| SC-006: Backward compatibility | Related to `status='Fixed'` changes |
