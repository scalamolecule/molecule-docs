---
title: "Transformation"
weight: 30
menu:
  main:
    parent: dev
    identifier: dev-transformation
---

# Macro transformation

Molecule performs a series of transformation of the molecules we write so that they in the end become a Datomic query:

The `m`olecule method transforms our source code _at compile time_ through a series of states:

1. Source code
2. Model AST
3. Query AST
4. Datomic query string

## 1. Source code

To find southern media communities we could make the following molecule
```
Community.name.`type`("twitter" or "facebook_page")
  .Neighborhood.District.region("sw" or "s" or "se")
```

## 2. Model AST

The source code of our molecule is then pattern matched in [Molecule2Model](https://github.com/scalamolecule/molecule/blob/master/core/src/main/scala/molecule/transform/Molecule2Model.scala) element by element in order to create an abstracted Model of `Atom`s and `Bond`s:

```
Model(List(
  Atom("Community", "name", "String", 1, VarValue, None),
  Atom("Community", "type_", "String", 1, Eq(List("twitter", "facebook_page")), Some(":Community.type/")),
  Bond("Community", "neighborhood", "Neighborhood"),
  Bond("Neighborhood", "district", "District"),
  Atom("District", "region_", "String", 1, Eq(List("sw", "s", "se")), Some(":District.region/")))
)
```
This simple [Model AST](https://github.com/scalamolecule/molecule/blob/master/core/src/main/scala/molecule/ast/model.scala) has shown to cover a surprising wide spectre of queries.

## 3. Query AST
Our model is then transformed in [Model2Query](https://github.com/scalamolecule/molecule/blob/master/core/src/main/scala/molecule/transform/Model2Query.scala) to a Query AST which is a little more elaborate:

```
Query(
  Find(List(
    Var("b"))),
  In(List(), List(
    Rule("rule1", List(Var("a")), List(
      DataClause(ImplDS, Var("a"), KW("Community", "type"), Val(":Community.type/twitter"), Empty))),
    Rule("rule1", List(Var("a")), List(
      DataClause(ImplDS, Var("a"), KW("Community", "type"), Val(":Community.type/facebook_page"), Empty))),
    Rule("rule2", List(Var("e")), List(
      DataClause(ImplDS, Var("e"), KW("District", "region"), Val(":District.region/sw"), Empty))),
    Rule("rule2", List(Var("e")), List(
      DataClause(ImplDS, Var("e"), KW("District", "region"), Val(":District.region/s"), Empty))),
    Rule("rule2", List(Var("e")), List(
      DataClause(ImplDS, Var("e"), KW("District", "region"), Val(":District.region/se"), Empty)))), List(DS)),
  Where(List(
    DataClause(ImplDS, Var("a"), KW("Community", "name"), Var("b"), Empty),
    RuleInvocation("rule1", List(Var("a"))),
    DataClause(ImplDS, Var("a"), KW("Community", "neighborhood", "Neighborhood"), Var("d"), Empty),
    DataClause(ImplDS, Var("d"), KW("Neighborhood", "district", "District"), Var("e"), Empty),
    RuleInvocation("rule2", List(Var("e")))))
)
``` 
As you see this [Query AST](https://github.com/scalamolecule/molecule/blob/master/core/src/main/scala/molecule/ast/query.scala) is tailored to Datomic.

In principle we should be able to use the same model to create other Query abstractions tailored to other database systems!...

## 4. Datomic query string

Finally Molecule transforms our Query AST in [Query2String](https://github.com/scalamolecule/molecule/blob/master/core/src/main/scala/molecule/transform/Query2String.scala) to a Datomic query text strings:

<pre>
[:find  ?b
 :in    $ %
 :where [?a :Community/name ?b]
        (rule1 ?a)
        [?a :Community/neighborhood ?d]
        [?d :Neighborhood/district ?e]
        (rule2 ?e)]

INPUTS:
List(
  1 datomic.db.Db@xxx
  2 [[(rule1 ?a) [?a :Community/type ":Community.type/twitter"]]
     [(rule1 ?a) [?a :Community/type ":Community.type/facebook_page"]]
     [(rule2 ?e) [?e :District/region ":District.region/sw"]]
     [(rule2 ?e) [?e :District/region ":District.region/s"]]
     [(rule2 ?e) [?e :District/region ":District.region/se"]]]
</pre>

All 3 transformations happen at compile time and therefore have no impact on the runtime performance.

See more examples of [transformation of the Seattle molecules](https://github.com/scalamolecule/molecule/blob/master/examples/src/test/scala/molecule/examples/seattle/SeattleTransformationTests.scala)...