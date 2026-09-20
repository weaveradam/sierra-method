---
template:
  id: https://www.modelware.io/sierra/system-analysis/connections
  name: "Connections"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Connections

View internal connections between system components.

```diagram
---
stylesheet:
  - selector: node
    style:
      stroke: red
      height: 100
      padding: 40
      layout:
        type: dagre
        ranksep: 150
        nodesep: 40
      label:
        refY: 10
        fill: vblack

  - selector: node.component
    style:
      fill: lightgreen

  - selector: node.subComponent
    style:
      fill: yellow

  - selector: edge
    style:
      stroke: red
      router:
        name: manhattan
        args:
          padding: 10
      label:
        font-size: 10
        fill: vblack
        text-anchor: middle
      label-body:
        fill: none

  - selector: port
    style:
      fill: var(--oml-static-background, #ffffff)
---
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX : <http://opencaesar.io/diagram#>

CONSTRUCT {
  ?component a :Node ;
             :class "component" .
   
  ?port a :Port ;
            :class "port" ;
            :parent ?component .

  ?subComponent a :Node ;
             :parent ?component ;
             :class "subComponent" .

  ?subPort a :Port ;
            :class "port" ;
            :parent ?subComponent .
 
  ?connection a :Edge ;
            :source ?srcPort ;
            :target ?tgtPort ;
            :text ?itemLabel .
}
WHERE {
  ?component component:hasPort ?port ;
     base:contains ?subComponent .

  ?subComponent component:hasPort ?subPort .

  ?connection a component:Connection ;
     oml:hasSource ?srcPort ;
     oml:hasTarget ?tgtPort ;
     component:transfers ?item .

    BIND(REPLACE(STR(?item), "^.*[#/]", "") AS ?itemLabel)
}
```

## Connection Direction Checks

Reject connections that run from an In port to an Out port.

```table-editor
---
columns: { this: { label: "Connection" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix oml: <http://opencaesar.io/oml#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:ConnectionShape
  a sh:NodeShape ;
  sh:targetClass component:Connection ;
  sh:property [
    sh:path oml:hasSource ;
    sh:name "Source" ;
    sh:class component:Port ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:order 1 ;
    oml:localReference true ;
  ] ;
  sh:property [
    sh:path oml:hasTarget ;
    sh:name "Target" ;
    sh:class component:Port ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:order 2 ;
    oml:localReference true ;
  ] ;
  sh:property [
    sh:path component:transfers ;
    sh:name "Transfers" ;
    sh:class base:Item ;
    sh:maxCount 1 ;
    sh:order 3 ;
  ] ;
  sh:property [
    sh:path base:description ;
    sh:name "Description" ;
    dash:editor dash:TextAreaEditor ;
    sh:maxCount 1 ;
    sh:order 4 ;
  ] ;
  sh:sparql [
    sh:message "A connection must not run from an In port to an Out port." ;
    sh:select """
      PREFIX component: <https://www.modelware.io/sierra/component#>
      PREFIX oml: <http://opencaesar.io/oml#>
      SELECT $this WHERE {
        $this oml:hasSource ?source ;
            oml:hasTarget ?target .
        OPTIONAL { ?source component:direction ?sourceDirection . }
        OPTIONAL { ?target component:direction ?targetDirection . }
        FILTER(
          STR(?sourceDirection) = "In" &&
          STR(?targetDirection) = "Out"
        )
      }
    """ ;
  ] ;
  .
```
