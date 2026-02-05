# SPFx Engine API Reference

This document provides a complete reference for the `SPFxProvisioningEngine` class, including initialization, execution, compliance checking, and error handling.

## Table of Contents

- [Overview](#overview)
- [SPFxProvisioningEngine Class](#spfxprovisioningengine-class)
- [Initialization](#initialization)
- [Execution](#execution)
- [Compliance Checking](#compliance-checking)
- [Real-Time Progress](#real-time-progress)
- [React Hooks](#react-hooks)
- [Error Handling](#error-handling)
- [Advanced Usage](#advanced-usage)
- [Best Practices](#best-practices)
- [Advanced API Reference](#advanced-api-reference)

---

## Overview

The `SPFxProvisioningEngine` is the main entry point for programmatic provisioning in SPFx solutions. It provides:

- **Schema validation**: Plans are validated against Zod schemas before execution
- **Depth-first execution**: Actions execute in order with scope propagation
- **Real-time progress**: Subscribe to execution snapshots
- **Compliance checking**: Drift detection without modifications
- **Cancellation**: Cooperative cancellation support

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   SPFxProvisioningEngine                    │
│  ├─ run(): Execute provisioning plan                        │
│  ├─ checkCompliance(): Drift detection                      │
│  ├─ subscribe(): Real-time progress updates                 │
│  └─ cancel(): Cooperative cancellation                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│               SharePointProvisioningEngine                  │
│  (Internal implementation for SharePoint operations)        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    PnPjs v4 (SPFI)                          │
│  SharePoint REST API operations                             │
└─────────────────────────────────────────────────────────────┘
```

---

## SPFxProvisioningEngine Class

### Import

```typescript
import { 
  SPFxProvisioningEngine,
  type SPFxProvisioningEngineOptions 
} from '@apvee/spfx-actionable-provisioning/provisioning';
```

### Constructor Options

```typescript
interface SPFxProvisioningEngineOptions {
  /** Root SPFI instance for SharePoint operations */
  spfi: SPFI;
  
  /** Initial SharePoint scope (SPFI is injected automatically) */
  initialScope?: SPScope;
  
  /** Provisioning plan (static JSON) */
  planTemplate: ProvisioningPlan;
  
  /** Logger instance */
  logger: Logger;
  
  /** Optional engine configuration */
  options?: EngineOptions;
}
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `run()` | `Promise<EngineSnapshot>` | Execute the provisioning plan |
| `checkCompliance(policy?)` | `Promise<ComplianceReport>` | Check drift without modifications |
| `subscribe(fn)` | `{ unsubscribe: () => void }` | Subscribe to real-time snapshots |
| `cancel()` | `void` | Request cooperative cancellation |

---

## Initialization

### Basic Initialization

```typescript
import { spfi, SPFx } from '@pnp/sp';
import { SPFxProvisioningEngine } from '@apvee/spfx-actionable-provisioning/provisioning';
import { createLogger, consoleSink } from '@apvee/spfx-actionable-provisioning/provisioning';
import { myPlan } from './provisioning-plan';

// Create SPFI instance with SPFx context
const sp = spfi().using(SPFx(this.context));

// Create logger
const logger = createLogger({ 
  level: 'debug', 
  sink: consoleSink 
});

// Initialize engine
const engine = new SPFxProvisioningEngine({
  spfi: sp,
  planTemplate: myPlan,
  logger
});
```

### With Initial Scope

When provisioning to a specific site different from the current context:

```typescript
import { Web } from '@pnp/sp/webs';

const targetSiteUrl = 'https://contoso.sharepoint.com/sites/target';

const engine = new SPFxProvisioningEngine({
  spfi: sp,
  planTemplate: myPlan,
  logger,
  initialScope: {
    web: Web([sp.web, targetSiteUrl])
  }
});
```

### With Engine Options

```typescript
const engine = new SPFxProvisioningEngine({
  spfi: sp,
  planTemplate: myPlan,
  logger,
  options: {
    // Continue execution after individual action failures
    continueOnActionError: false,
    
    // Enable permission preflight checks
    enablePreflightPermissions: true
  }
});
```

---

## Execution

### Running Provisioning

The `run()` method executes the provisioning plan and returns a final snapshot:

```typescript
const engine = new SPFxProvisioningEngine({
  spfi: sp,
  planTemplate: myPlan,
  logger
});

try {
  const snapshot = await engine.run();
  
  if (snapshot.status === 'completed') {
    console.log('Provisioning completed successfully');
    console.log(`Total actions: ${snapshot.summary.total}`);
    console.log(`Succeeded: ${snapshot.summary.succeeded}`);
  } else if (snapshot.status === 'failed') {
    console.error('Provisioning failed:', snapshot.error);
  } else if (snapshot.status === 'cancelled') {
    console.log('Provisioning was cancelled');
  }
} catch (error) {
  console.error('Engine error:', error);
}
```

### Engine Status Lifecycle

```
idle → preprocessing → preflightPermissions → running → completed
                                                      ↘ failed
                                                      ↘ cancelled
```

| Status | Description |
|--------|-------------|
| `idle` | Engine initialized, ready to execute |
| `preprocessing` | Expanding templates and validating plan |
| `preflightPermissions` | Checking permissions before execution |
| `running` | Executing actions |
| `completed` | All actions completed successfully |
| `failed` | Execution stopped due to an error |
| `cancelled` | Execution was cancelled by user |

### Engine Snapshot

The snapshot contains complete execution state:

```typescript
interface EngineSnapshot<TResult> {
  /** Current engine status */
  status: EngineStatus;
  
  /** Execution summary */
  summary: {
    total: number;
    succeeded: number;
    failed: number;
    skipped: number;
    pending: number;
  };
  
  /** Per-action trace entries */
  trace: TraceEntry<TResult>[];
  
  /** Error (if status is 'failed') */
  error?: Error;
  
  /** Timestamp of this snapshot */
  timestamp: string;
}
```

### Trace Entries

Each action produces a trace entry:

```typescript
interface TraceEntry<TResult> {
  /** Action path (e.g., "0", "0.1", "0.1.2") */
  path: ActionPath;
  
  /** Action verb */
  verb: string;
  
  /** Trace status */
  status: 'pending' | 'running' | 'succeeded' | 'failed' | 'skipped';
  
  /** Action result (if succeeded) */
  result?: TResult;
  
  /** Error (if failed) */
  error?: Error;
  
  /** Start timestamp */
  startedAt?: string;
  
  /** End timestamp */
  endedAt?: string;
}
```

---

## Compliance Checking

### Basic Compliance Check

Compliance checking evaluates whether the current SharePoint state matches the plan's desired state **without making any changes**:

```typescript
const engine = new SPFxProvisioningEngine({
  spfi: sp,
  planTemplate: myPlan,
  logger
});

const report = await engine.checkCompliance();

console.log(`Overall: ${report.overall}`); // 'compliant' | 'warning' | 'non_compliant'
console.log(`Items checked: ${report.items.length}`);

for (const item of report.items) {
  if (item.outcome === 'non_compliant') {
    console.log(`Drift detected: ${item.path} - ${item.details}`);
  }
}
```

### With Compliance Policy

Configure how unverifiable actions are handled:

```typescript
const report = await engine.checkCompliance({
  // How to treat actions that cannot be verified
  treatUnverifiableAs: 'warning' // or 'ignore'
});
```

### Compliance Report Structure

```typescript
interface ComplianceReport {
  /** Overall compliance status */
  overall: 'compliant' | 'warning' | 'non_compliant';
  
  /** Per-action compliance items */
  items: ComplianceItem[];
}

interface ComplianceItem {
  /** Action path */
  path: ActionPath;
  
  /** Compliance outcome */
  outcome: 'compliant' | 'non_compliant' | 'unverifiable' | 'ignored';
  
  /** Human-readable details */
  details?: string;
}
```

### Compliance Outcomes

| Outcome | Description |
|---------|-------------|
| `compliant` | Current state matches desired state |
| `non_compliant` | Current state differs from desired state |
| `unverifiable` | Unable to determine compliance (e.g., API limitations) |
| `ignored` | Action was ignored per policy |

---

## Real-Time Progress

### Subscribing to Updates

Subscribe to receive snapshots as execution progresses:

```typescript
const engine = new SPFxProvisioningEngine({
  spfi: sp,
  planTemplate: myPlan,
  logger
});

// Subscribe before running
const { unsubscribe } = engine.subscribe((snapshot) => {
  console.log(`Status: ${snapshot.status}`);
  console.log(`Progress: ${snapshot.summary.succeeded}/${snapshot.summary.total}`);
  
  // Find currently running action
  const running = snapshot.trace.find(t => t.status === 'running');
  if (running) {
    console.log(`Running: ${running.verb} at path ${running.path}`);
  }
});

// Run the engine
await engine.run();

// Cleanup
unsubscribe();
```

### React Hook Integration

For React components, use the provided hooks for lifecycle-safe engine management.

---

## React Hooks

The library provides React hooks for managing provisioning engine state in components.

### useSPFxProvisioningEngine

The primary hook for provisioning operations in React components:

```tsx
import { 
  useSPFxProvisioningEngine,
  type UseSPFxProvisioningEngineOptions,
  type UseSPFxProvisioningEngineReturn
} from '@apvee/spfx-actionable-provisioning/provisioning-ui';
```

#### Options

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `context` | `BaseComponentContext` | ✓ | SPFx component context |
| `planTemplate` | `ProvisioningPlan` | ✓ | Provisioning plan to execute |
| `logger` | `Logger` | ✓ | Logger instance |
| `targetSiteUrl` | `string` | - | Target site URL (defaults to current site) |
| `initialScope` | `SPScope` | - | Initial scope for the engine |
| `resetKey` | `number` | - | Increment to reset engine state |
| `engineOptions` | `EngineOptions` | - | Additional engine configuration |

#### Return Value

```typescript
type UseSPFxProvisioningEngineReturn = Readonly<{
  /** Current engine snapshot (undefined until first update) */
  snapshot: EngineSnapshotTyped<ProvisioningResultLight> | undefined;
  
  /** Start provisioning execution */
  run: () => Promise<EngineSnapshot>;
  
  /** Cancel current operation */
  cancel: () => void;
  
  /** Run compliance check */
  checkCompliance: (policy?: CompliancePolicy) => Promise<ComplianceReport>;
}>;
```

#### Example

```tsx
import { useSPFxProvisioningEngine } from '@apvee/spfx-actionable-provisioning/provisioning-ui';
import type { Logger, ProvisioningPlan } from '@apvee/spfx-actionable-provisioning/provisioning';

const MyComponent: React.FC<{ context: WebPartContext }> = ({ context }) => {
  const { snapshot, run, cancel, checkCompliance } = useSPFxProvisioningEngine({
    context,
    targetSiteUrl: 'https://contoso.sharepoint.com/sites/target',
    planTemplate: myPlan,
    logger,
    resetKey: 0 // Increment to reset engine state
  });

  const isRunning = snapshot?.status === 'running';
  
  return (
    <div>
      <p>Status: {snapshot?.status}</p>
      <p>Progress: {snapshot?.summary.succeeded}/{snapshot?.summary.total}</p>
      
      <Button onClick={run} disabled={isRunning}>
        Run Provisioning
      </Button>
      <Button onClick={cancel} disabled={!isRunning}>
        Cancel
      </Button>
    </div>
  );
};
```

### useProvisioningDerivedState

A utility hook that derives UI-friendly state from engine snapshots:

```tsx
import { 
  useProvisioningDerivedState,
  type ProvisioningDerivedState
} from '@apvee/spfx-actionable-provisioning/provisioning-ui';
```

#### Usage

```tsx
const { snapshot } = useSPFxProvisioningEngine({ /* ... */ });

const { summary, logEntries } = useProvisioningDerivedState(snapshot);

// summary: ProvisioningUiSummary | undefined
// logEntries: ReadonlyArray<ProvisioningLogEntry>
```

#### Return Value

```typescript
type ProvisioningDerivedState = Readonly<{
  /** UI-friendly summary with counts and status */
  summary?: ProvisioningUiSummary;
  
  /** Log entries for display in a log panel */
  logEntries: ReadonlyArray<ProvisioningLogEntry>;
}>;
```

This hook memoizes the derived state to prevent unnecessary re-renders.

---

## Error Handling

### Execution Errors

```typescript
const snapshot = await engine.run();

if (snapshot.status === 'failed') {
  // Engine-level error
  console.error('Engine error:', snapshot.error?.message);
  
  // Find failed actions
  const failedActions = snapshot.trace.filter(t => t.status === 'failed');
  for (const action of failedActions) {
    console.error(`Action ${action.verb} at ${action.path} failed:`, action.error);
  }
}
```

### Action-Level Errors

Each action may produce its own error in the trace:

```typescript
for (const entry of snapshot.trace) {
  if (entry.status === 'failed') {
    console.error(`${entry.verb} failed:`, entry.error?.message);
  }
}
```

### Skipped Actions

Actions may be skipped for various reasons:

```typescript
for (const entry of snapshot.trace) {
  if (entry.status === 'skipped' && entry.result) {
    const result = entry.result as ProvisioningResultLight;
    if (result.outcome === 'skipped') {
      console.log(`${entry.verb} skipped: ${result.reason}`);
      // Reasons: 'not_found', 'already_exists', 'no_changes', 'missing_prerequisite', 'unsupported'
    }
  }
}
```

### Skip Reasons

| Reason | Description |
|--------|-------------|
| `not_found` | Target resource doesn't exist (for modify/delete) |
| `already_exists` | Resource already exists (for create) |
| `no_changes` | No differences between current and desired state |
| `missing_prerequisite` | Required parent resource is missing |
| `unsupported` | Operation not supported for this resource type |

---

## Advanced Usage

### Accessing the SharePoint Engine

For advanced scenarios, access the underlying SharePoint engine:

```typescript
const spEngine = engine.sharepointEngine;

// Use SharePoint-specific features
```

### Custom Logger

Create a custom logger for specific logging needs:

```typescript
import { createLogger, type LogSink } from '@apvee/spfx-actionable-provisioning/provisioning';

// Custom sink that sends logs to Application Insights
const appInsightsSink: LogSink = (level, message, context) => {
  appInsights.trackTrace({
    message,
    severityLevel: level === 'error' ? 3 : level === 'warn' ? 2 : 1,
    properties: context
  });
};

const logger = createLogger({
  level: 'info',
  sink: appInsightsSink,
  scope: { component: 'Provisioning' }
});
```

### Timeout and Retry

Implement timeout and retry logic:

```typescript
const runWithTimeout = async (engine: SPFxProvisioningEngine, timeoutMs: number) => {
  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => {
      engine.cancel();
      reject(new Error('Provisioning timed out'));
    }, timeoutMs);
  });

  return Promise.race([engine.run(), timeoutPromise]);
};

// Usage
try {
  const snapshot = await runWithTimeout(engine, 60000); // 60 second timeout
} catch (error) {
  console.error('Provisioning failed or timed out:', error);
}
```

---

## Best Practices

### Engine Lifecycle

1. **Create engine per operation**: Don't reuse engine instances across multiple runs
2. **Always unsubscribe**: Clean up subscriptions when components unmount
3. **Handle all statuses**: Check for `completed`, `failed`, and `cancelled`

### Performance

1. **Use scope inheritance**: Let subactions inherit parent scope handles
2. **Avoid redundant operations**: Check compliance before provisioning
3. **Log appropriately**: Use `'info'` level in production, `'debug'` for development

### Error Recovery

1. **Implement retry logic**: For transient failures (network, throttling)
2. **Use compliance checking**: Verify state after failures before retrying
3. **Persist state**: Store provisioning state in web part properties

### Security

1. **Validate plans**: Never execute user-provided plans without validation
2. **Minimum permissions**: Request only necessary API permissions
3. **Audit logging**: Log provisioning operations for compliance

---

## Advanced API Reference

This section documents utility functions and types for advanced use cases.

### Utility Functions

Import from `@apvee/spfx-actionable-provisioning/provisioning-ui`:

#### buildProvisioningLogEntriesFromSnapshot

```typescript
function buildProvisioningLogEntriesFromSnapshot(
  snapshot: EngineSnapshotTyped<ProvisioningResultLight> | undefined
): ReadonlyArray<ProvisioningLogEntry>;
```

Converts an engine snapshot into an array of log entries for UI display.

#### buildProvisioningUiSummary

```typescript
function buildProvisioningUiSummary(
  snapshot: EngineSnapshotTyped<ProvisioningResultLight> | undefined
): ProvisioningUiSummary | undefined;
```

Extracts a UI-friendly summary from an engine snapshot.

#### buildComplianceLogEntriesFromReport

```typescript
function buildComplianceLogEntriesFromReport(
  report: ComplianceReport | undefined
): ReadonlyArray<ComplianceLogEntry>;
```

Converts a compliance report into log entries for display.

#### buildComplianceLogEntriesFromTrace

```typescript
function buildComplianceLogEntriesFromTrace(
  trace: ComplianceTrace | undefined
): ReadonlyArray<ComplianceLogEntry>;
```

Converts a compliance trace into log entries for display.

### Model Types

#### ProvisioningUiSummary

```typescript
type ProvisioningUiSummary = Readonly<{
  total: number;
  succeeded: number;
  failed: number;
  skipped: number;
  pending: number;
  status: EngineStatus;
}>;
```

#### ProvisioningUiProgress

```typescript
type ProvisioningUiProgress = Readonly<{
  current: number;
  total: number;
  percentage: number;
}>;
```

#### ProvisioningLogEntry

```typescript
type ProvisioningLogEntry = BaseLogEntry & Readonly<{
  kind: 'provisioning';
  status: ProvisioningLogStatus;
  verb: string;
  path: ActionPath;
  result?: ProvisioningResultLight;
  error?: Error;
}>;
```

#### ProvisioningLogStatus

```typescript
type ProvisioningLogStatus = 'pending' | 'running' | 'succeeded' | 'failed' | 'skipped';
```

#### ComplianceLogEntry

```typescript
type ComplianceLogEntry = BaseLogEntry & Readonly<{
  kind: 'compliance';
  outcome: ComplianceOutcome;
  verb: string;
  path: ActionPath;
  details?: string;
}>;
```

#### TemplateAppliedState

```typescript
type TemplateAppliedState = 'applied' | 'not-applied' | 'unknown';
```

Used to track whether a provisioning template has been applied to a site.

---

## Related Documentation

- [Introduction](./introduction.md) - Getting started guide
- [Provisioning Schema](./provisioning-schema.md) - Plan structure and actions
- [ProvisioningDialog](./provisioning-dialog.md) - Dialog component for UI
- [Property Pane Fields](./property-pane-fields.md) - Property pane integration
