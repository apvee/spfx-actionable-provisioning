<!--
  SYNC IMPACT REPORT (v1.0.0)
  ═══════════════════════════════════════════════════════════════
  Version Change: N/A → 1.0.0 (initial ratification)
  
  Added Sections:
    - Core Principles (5 principles)
    - Technology Stack
    - Development Workflow
    - Governance
  
  Templates Requiring Updates:
    ✅ plan-template.md - Constitution Check section compatible
    ✅ spec-template.md - Requirements structure compatible
    ✅ tasks-template.md - Phase structure compatible
  
  Follow-up TODOs: None
  ═══════════════════════════════════════════════════════════════
-->

# @apvee/spfx-actionable-provisioning Constitution

## Core Principles

### I. Schema-First Architecture

Every provisioning action MUST be validated by a Zod schema before execution. This ensures:
- Runtime validation prevents malformed plans from executing
- TypeScript types are inferred directly from schemas (single source of truth)
- Schema definitions live in `src/provisioning/types/` with corresponding catalog entries
- New action types require both schema definition AND catalog registration

**Rationale**: Schema validation catches configuration errors before they reach SharePoint,
preventing partial provisioning failures and ensuring predictable execution.

### II. Type Safety

Full TypeScript support is MANDATORY throughout the codebase:
- `strict: true` in tsconfig.json; no `any` types without explicit justification
- All public APIs MUST export proper TypeScript types
- Zod schemas MUST use `.infer<>` for type derivation
- PnPjs handles retain their original types through scope chain

**Rationale**: Type safety enables IDE support, catches errors at compile time, and ensures
the library integrates seamlessly with other TypeScript SPFx projects.

### III. Scope-Based Execution

The engine uses a handle-based scope chain for execution context:
- Each action receives `scopeIn` containing PnPjs handles (`spfi`, `site`, `web`, `list`)
- Actions produce `scopeDelta` merged into `scopeOut`
- Subactions inherit parent scope automatically
- Non-plain objects (PnPjs handles) are treated as atomic ("last wins" merge)

**Rationale**: Handle-based scope eliminates redundant API calls, provides natural context
inheritance, and avoids storing URL strings that require re-resolution.

### IV. Declarative Plans

Provisioning plans MUST be declarative JSON defining desired state:
- Plans describe WHAT to provision, NOT HOW
- Parameters use `{parameter:KeyName}` syntax for dynamic substitution
- Actions support nested `subactions` for hierarchical execution
- Plans MUST include `schemaVersion` and `version` fields

**Rationale**: Declarative plans are portable, auditable, version-controllable, and can be
validated before execution. They separate intent from implementation.

### V. Compliance Checking

Built-in drift detection MUST be supported without modifying the environment:
- `checkCompliance()` runs validation without making changes
- Compliance results include detailed action-level status
- Preflight permission checks prevent unauthorized provisioning attempts
- All compliance operations are read-only

**Rationale**: Compliance checking enables "what-if" scenarios, audit trails, and safe
validation in production environments without risk of accidental modifications.

## Technology Stack

The following technology constraints MUST be followed:
- **Runtime**: SPFx 1.18+ (SharePoint Framework)
- **Language**: TypeScript 5.x with strict mode
- **SharePoint API**: PnPjs v4 exclusively (no REST/CSOM direct calls)
- **Validation**: Zod v4 for schema definitions
- **UI Components**: FluentUI React (when UI is required)
- **Build**: Gulp-based SPFx toolchain
- **Package Manager**: npm (lockfile committed)

**Rationale**: Consistent technology stack ensures maintainability, reduces onboarding
friction, and aligns with Microsoft's SPFx best practices.

## Development Workflow

All contributions MUST follow these workflow requirements:
- **Barrel Exports**: Each module (`core`, `provisioning`, `provisioning-ui`) exposes public
  API through `index.ts` barrel files; internal modules are not exported
- **Documentation**: Public APIs MUST have JSDoc comments; new features MUST update
  corresponding docs in `/docs/`
- **Backwards Compatibility**: Breaking changes require MAJOR version bump and migration guide
- **Logging**: All engine operations MUST use the `Logger` interface (no `console.log`)
- **Error Handling**: Errors MUST be typed and include actionable context

**Rationale**: Clear workflow guidelines ensure consistent code quality, enable seamless
library consumption, and maintain upgrade paths for consumers.

## Governance

This constitution supersedes all other development practices. Amendments require:
1. Documented justification for the change
2. Impact assessment on existing features
3. Version bump following semantic versioning:
   - MAJOR: Backward-incompatible principle changes
   - MINOR: New principles or significant expansions
   - PATCH: Clarifications and wording improvements

All pull requests MUST verify compliance with these principles. Complexity beyond these
guidelines MUST be explicitly justified in PR descriptions.

**Version**: 1.0.0 | **Ratified**: 2026-03-16 | **Last Amended**: 2026-03-16
