---
ontology: https://fireforce6.github.io/mission-control/bundle
---

```compose
template: https://www.modelware.io/sierra/context-analysis/dashboard
```

## Objective Allocation Coverage

**Engineering question:** For each mission objective, have all of its required capabilities been allocated to at least one modeled operational system?

**Evidence:** The matrix shows each objective's required capabilities. A `1` means at least one system-level operational entity is assigned that required capability; `0` means the required capability has no system allocation. Capabilities not required by an objective are omitted.

```compose
template: https://www.modelware.io/sierra/context-analysis/fire-force-capability-coverage
```

**Interpretation:** All eight required objective/capability pairs currently have a system allocation. There are no missing allocations in this view; a future `0` would identify a required capability that no modeled operational system realizes.