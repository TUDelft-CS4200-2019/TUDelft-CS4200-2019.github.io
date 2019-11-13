---
layout: page
title: "Lab 7: Type Analysis"
excerpt: "Lab 7: Type Analysis"
tags: ["project"]
context: project
subcontext: ms2
---

{% include _toc.html %}

In this lab, you define name bindings and corresponding constraints for MiniJava in Statix. The
concepts you are going to use in Statix are described in the following papers:

1. H. van Antwerpen, C. B. Poulsen, A. Rouvoet, E. Visser: [Scopes as Types](https://researchr.org/publication/AntwerpenPRV18), OOPSLA 2018
2. P. Neron, A. Tolmach, E. Visser, G. Wachsmuth: [A Theory of Name Resolution](https://researchr.org/publication/TUD-SERG-2015-001), ESOP 2015
3. H. van Antwerpen, P. Neron, A. Tolmach, E. Visser, G. Wachsmuth: [A Constraint Language for Static Semantic Analysis based on Scope Graphs](https://researchr.org/publication/AntwerpenNTVW16), PEPM 2016

Update your Spoofax installation for this lab, to get the latest bug fixes.
See the [update instructions](/documentation/spoofax#updating).
{: .notice .notice-info}

## Overview

### Objectives

Specify type analysis for MiniJava in Statix. The specification should include:

1. Name binding constraints for
  * method calls.
2. Typing constraints for
  * integer and boolean literals,
  * unary and binary expressions,
  * variable and field references,
  * object creation,
  * `this` expressions,
  * method calls,
  * and the subtyping relation on classes.

### Submission

You need to submit your MiniJava project with a merge request against branch `assignment-7-submission` on GitLab.
The [Git documentation](/documentation/git.html#submitting-an-assignment) explains how to file such a request.

The deadline for submission is November 23, 2019, 23:59.
{: .notice .notice-warning}

### Grading

You can earn up to 100 points for the correctness of your type analysis. Therefore, we run several
test cases against your implementation. You earn points, when your implementation passes test
cases. The total number of points depends on how many test cases you pass in each of the following
groups:

* name binding (20 points)
  * method declarations
  * method references
* typing rules (35 points)
  * literals (4 points)
  * unary expressions (5 points)
  * binary expressions (8 points)
  * variable and field references (4 points)
  * object creation (4 point)
  * `this` expression (5 points)
  * method call (5 points)
* constraints (45 points)
  * expressions (10 points)
  * statements (10 points)
  * subtyping in assignments, return expressions, method calls (10 points)
  * cyclic inheritance (2 points)
  * method overloading and method overriding (10 points)
  * main class usage (3 points)

### Early Feedback

We provide early feedback for your language implementation. This feedback gives you an indication
which parts of your name and type analysis might still be wrong. It includes a summary on how many
tests you pass and how many points you earn by passing them. You have 3 early feedback attempts.

## Detailed Instructions

### Git Repository

You continue with your work from the previous assignment. See the
[Git documentation](/documentation/git.html#continue-from-previous-assignment) on how to create the
`assignment-7-develop` branch from your previous work.

For the grading to work correctly, you have to pull some changes from the template.
See the [instructions](/git.html/#pulling-in-changes-from-template) on how to do that.
{: .notice .notice-info}

### Example specification

A complete example specification for the simply typed lambda calculus can be found at [STLC](lab5-example.html). Furthermore, we would recommend looking at the [Tiger Statix implementation](https://github.com/MetaBorgCube/metaborg-tiger/blob/master/org.metaborg.lang.tiger.statix/trans/static-semantics.stx).

### Declaring Types

In this lab we use the _scopes as types_ idea to model class types. Instead of using the syntactic
type constructors from the syntax directly, we introduce semantic types. These are the types that we
use as the types of declarations and AST nodes. The types you should use are given by the following
signature, which you should add to your specification:

    signature
      sorts TYPE constructors
        CLASS    : scope -> TYPE
        INT      : TYPE
        BOOL     : TYPE
        INTARRAY : TYPE

We use _scopes as types_ for classes, where the class is identified by its scope. Sometimes it is
useful (and for this lab required for grading) to map the scope back to the original class
name. Therefore, you must define a relation that associates the class name with the class scope:

    signature
      relations
        className : scope -> ID
        
In the rule for class declarations, use the constraint `!className[x] in s_cls`, where `x` is the
class name, and `s_cls` is the class scope, to store the class name. *If you don't do this, some of
the grading tests will fail!*

You also need to add a type constructor for methods. This type should have two arguments, the return
type, and a list of argument types of the method. We do not test for this type directly, so you can
choose its name yourself.

### Predicate Rules for Typing

Building on the predicate rules of lab 5, we need to extend the rules for AST nodes that have types
with an extra type parameter. Instead of adding this as a regular parameter to the predicate, we
usually prefer to use the functional syntax for these. For example, the predicate for typing
expressions would get the following signature and rules:

    expOk : scope * Exp -> TYPE
    
    expOk(s, True()) = BOOL().

When using the function syntax, the predicate calls have to be used in term positions. For example,
in an equality constraint like `expOk(s, e) == T`. Alternatively, you can use regular predicates and
add an extra parameter. In that case, the signature and rules look like this:

    expOk : scope * Exp * TYPE
    
    expOk(s, True(), BOOL()).

In the latter case, the predicate is used as a predicate, not a term, and the usage would be
`expOk(s, e, T)`. Although we usually prefer the functional form, both are equivalent, and you are
free to choose either of them. Note that the functional form is translated to the predicate form,
which you can see using the `Spoofax > Syntax > Format Normalized AST` menu.

The type is not automatically associated with the AST node. We do this be setting the `type`
property on an AST node, using the constraint `@e.type := T`. For example, the rule for `true` could
be written as:

    expOk(s, e@True()) = T@BOOL() :- @e.type := T.

The convenience syntax `v@t` is inline notation for `v == t`, for a variable `v` and a term `t`.

### Typing Declarations

In this lab we need to associate types with the declarations in the scope graph. Therefore, we
replace the `decl` and `scopeOf` relations with a new `type` relation with the following signature:

    signature
      relations
        type : occurrence -> TYPE

Declarations that were introduced with `s -> OCCURRENCE` must now add a type as well, using the
syntax `s -> OCCURRENCE with type TYPE`. Similarly, resolution queries such as `Var{x} in s |-> [(_,
Var{x'})]`, must now query the `type` relation as `type of Var{x} in s |-> [(_, (Var{x'}, T))]`.

### Subtyping

MiniJava supports subtyping between classes. To support this, you will have to implement a predicate
to check whether one type is a subtype of another type. This predicate must be used instead of type
equality wherever subtyping is allowed.

Subtyping between classes can be checked by using the structure of the scope graph. For this you can
use a query that does not query a specific relation, but only considers the final scope of a
path. The syntax for that is:

    query () filter ... and { s :- ... } min ... and { s1, s2 :- ... } in s |-> ps

where the result `ps` has type `list((path * scope))`.

### Type-dependent Name Resolution

Using types we can now implement resolution of method names. Use the class types to resolve method
names in the right scope. The _scopes as types_ representation of class types should make this
easy. Do not forget to add the `@x.ref := x'` property on method references, to ensure editor
resolution and testing work.

Remember that you can check if method resolution works, by *Ctrl/Cmd + click* on a method call in an
example program.
{: .notice .notice-info}

### Overriding

Finally, check if overriding is done correctly. If a method overrides a method from the super class,
its parameter types must be the same. Also check that the return type is correct. Be careful, it
does not have to be the same, but it needs to be co-variant! Incorrect overrides (including
overloads) should result in an error.
