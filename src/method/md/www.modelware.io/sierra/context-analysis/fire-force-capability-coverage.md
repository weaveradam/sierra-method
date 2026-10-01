---
template:
  id: https://www.modelware.io/sierra/context-analysis/fire-force-capability-coverage
  name: "Fire Force Objective Allocation Coverage"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Objective / Required Capability Allocation

Each row is a mission objective and each column is a capability that objective requires. A cell is 1 when at least one system-level operational entity is assigned that required capability, and 0 when no such allocation is modeled. Capability pairs the objective does not require are not displayed, so they cannot be mistaken for gaps.

```matrix
---
rowColumnLabel: Objective / Required Capability
stylesheet:
  - selector: cell [Number(value) === 0]
    style:
      background-color: mistyrose
  - selector: cell [Number(value) === 1]
    style:
      background-color: lightgreen
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>

SELECT ?row ?column (IF(COUNT(?system) > 0, 1, 0) AS ?value)
WHERE {
  ?row a mission:Objective ;
       mission:requires ?column .
  OPTIONAL {
    ?column entity:isAssignedTo ?system .
    GRAPH <https://fireforce6.github.io/mission-control/operational-analysis/entities> {
      ?system a entity:Entity .
    }
  }
}
GROUP BY ?row ?column
ORDER BY ?row ?column
```