---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Assignment 5: Analysis Layer

## 1. Conformance: stakeholder concerns

**Engineering question:** Does every modeled stakeholder express at least one concern?

**Evidence:** The query returns every stakeholder with a PASS or FAIL result.

```table
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>

SELECT ?stakeholder ?concern ?status
WHERE {
  ?stakeholder a stakeholder:Stakeholder .
  OPTIONAL { ?stakeholder stakeholder:expresses ?concern }
  BIND(IF(BOUND(?concern), "PASS", "FAIL") AS ?status)
}
ORDER BY ?stakeholder
```

**Interpretation:** The current result has five rows, all PASS. This tests a Sierra modeling rule as a conformance check; a stakeholder with no expressed concern would remain visible and return FAIL because the outer pattern starts from all stakeholders.

## 2. Near miss: partial objective allocation

This review query looks for objectives that have at least one required capability allocated to an entity, but not all of their required capabilities. It identifies objectives close to complete allocation without treating that heuristic as a formal Sierra constraint.

```table
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>

SELECT ?objective ?required ?allocated
WHERE {
  {
    SELECT ?objective
           (COUNT(DISTINCT ?capability) AS ?required)
           (COUNT(DISTINCT ?assignedCapability) AS ?allocated)
    WHERE {
      ?objective a mission:Objective ; mission:requires ?capability .
      OPTIONAL {
        ?capability entity:isAssignedTo ?entity .
        ?entity a entity:Entity .
        BIND(?capability AS ?assignedCapability)
      }
    }
    GROUP BY ?objective
  }
  FILTER(?allocated > 0 && ?allocated < ?required)
}
ORDER BY ?objective
```

**Interpretation:** No rows means there are no partial allocations under this definition. The full counts are 1/1 for O1, 3/3 for O2, 2/2 for O3, and 2/2 for O4, so every required capability currently has at least one entity assignment.

## 3. Orphan: concern without a mission trace

**Engineering question:** Is any expressed stakeholder concern missing a path through a pursued objective?

```table
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>

SELECT ?stakeholder ?concern
WHERE {
  ?stakeholder a stakeholder:Stakeholder ;
               stakeholder:expresses ?concern .
  FILTER NOT EXISTS {
    ?objective a mission:Objective ;
               mission:isDerivedFrom ?concern .
    ?mission a mission:Mission ;
             mission:pursues ?objective .
  }
}
ORDER BY ?stakeholder
```

**Interpretation:** The result is empty: all five expressed concerns are traced to at least one objective pursued by the modeled mission. If a stakeholder concern were not connected to a pursued objective, it would appear here as an orphan requiring review.

## 4. Coverage: objective allocation of required capabilities

The reusable matrix uses mission objectives as rows and only the capabilities each objective requires as columns. For each required pair, it checks whether at least one system-level operational entity is assigned that capability. This avoids comparing stakeholders, systems, and components as if they were peers, and avoids showing zeros for objective/capability pairs that are not required. A zero now has a specific meaning: a required capability has no allocation to a modeled operational system. The same template is invoked from the Context Dashboard.

```compose
template: https://www.modelware.io/sierra/context-analysis/fire-force-capability-coverage
```

**Interpretation:** The live matrix has eight required objective/capability pairs, all with value 1. No required capability is currently missing an allocation to an operational system. If a required capability lost all system allocations, its cell would become 0 and identify the objective affected.

## 5. View graph: concern to mission objective

This `CONSTRUCT` result shapes the graph around the traceability path rather than returning every fact in the source model.

```graph
---
layout:
  mode: force
  running: true
  fit: true
  padding: 24
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>

CONSTRUCT {
  ?stakeholder a stakeholder:Stakeholder .
  ?concern a stakeholder:Concern .
  ?objective a mission:Objective .
  ?mission a mission:Mission .
  ?stakeholder stakeholder:expresses ?concern .
  ?objective mission:isDerivedFrom ?concern .
  ?mission mission:pursues ?objective .
}
WHERE {
  ?stakeholder a stakeholder:Stakeholder ;
               stakeholder:expresses ?concern .
  ?objective a mission:Objective ;
             mission:isDerivedFrom ?concern .
  ?mission a mission:Mission ;
           mission:pursues ?objective .
}
```

## Scripted analysis: query, compute, render

The script queries required capabilities per objective, counts how many of those capabilities are assigned to at least one system-level operational entity, computes the completion percentage, and renders one comparable row per objective. Its denominator is each objective's actual required-capability count, not every capability in the vocabulary.

```python
import html

result = await query("""
  PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
  PREFIX mission: <https://www.modelware.io/sierra/mission#>
  PREFIX entity: <https://www.modelware.io/sierra/entity#>

  SELECT ?objective ?objectiveLabel
         (COUNT(DISTINCT ?capability) AS ?required)
         (COUNT(DISTINCT ?allocatedCapability) AS ?allocated)
  WHERE {
    ?objective a mission:Objective ;
               mission:requires ?capability .
    OPTIONAL { ?objective rdfs:label ?objectiveLabel }
    OPTIONAL {
      ?capability entity:isAssignedTo ?system .
      GRAPH <https://fireforce6.github.io/mission-control/operational-analysis/entities> {
        ?system a entity:Entity .
      }
      BIND(?capability AS ?allocatedCapability)
    }
  }
  GROUP BY ?objective ?objectiveLabel
  ORDER BY ?objective
""")

rows = result["rows"]
table_rows = []
required_total = 0
allocated_total = 0
for row in rows:
    objective = row.get("objectiveLabel") or row["objective"].rsplit("#", 1)[-1].rsplit("/", 1)[-1]
    required = int(row["required"])
    allocated = int(row["allocated"])
    percent = 100 * allocated / required if required else 0
    required_total += required
    allocated_total += allocated
    table_rows.append(
        f"<tr><td>{html.escape(objective)}</td><td>{allocated}/{required}</td>"
        f"<td>{percent:.0f}%</td></tr>"
    )

total_percent = 100 * allocated_total / required_total if required_total else 0
display(
    f"<p><strong>{allocated_total}/{required_total}</strong> required objective/capability "
    f"pairs are allocated ({total_percent:.1f}% coverage).</p>"
    '<table class="oml-md-table"><thead><tr><th>Objective</th>'
    '<th>Required capabilities allocated</th><th>Coverage</th></tr></thead>'
    f"<tbody>{''.join(table_rows)}</tbody></table>"
)
```