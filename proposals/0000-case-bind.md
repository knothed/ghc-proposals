---
author: David Knothe, John Ericson, Sebastian Graf
date-accepted: ""
ticket-url: ""
implemented: ""
---

This proposal is [discussed at this pull request](https://github.com/ghc-proposals/ghc-proposals/pull/0>).
**After creating the pull request, edit this file again, update the number in
the link, and delete this bold sentence.**

# Case bind

Provide a more concise way to handle failure patterns in do notation.

## Motivation

Suppose I want to write imperative-style monadic code with manual error handling using pattern-matching. "Manual", because the `fail` desugar doesn't do what I want and "using pattern-matching" because I'm not using exceptions or similar.

I could do it like this:

```hs
do
  res0 <- action0
  case res0 of
    Good0... -> do
      res1 <- action1
      case res1 of
        Good1... ->
          ...
        Bad1_0... -> ...
        Bad1_1... -> ...
    Bad0_0... -> ...
    Bad0_1... -> ...
```

Notice how the indentation grows at every failure, and also the earlier good and bad patterns are quite far apart.

Another possibility is:

```hs
do
  res0 <- action0
  TupleVars0 <- case res0 of
    Good0... -> pure TupleVars0
    Bad0_0... -> ...
    Bad0_1... -> ...
  res1 <- action1
  TupleVars1 <- case res1 of
    Good1... -> pure TupleVars1
    Bad1_1... -> ...
    Bad1_1... -> ...
  ...
```

This solves the rightward drift and distant alternatives problems, but at the cost of making the user manually construct and destruct tuples of all the variables they bound in the "good case". This is tedious, and makes the code harder to modify.

Clearly, both styles have drawbacks.


## Proposed Change Specification

We introduce a new sugar in `do`-notation, using the Haskell Report 2010's non-terminals:

```hs
stmt → case pat <- exp of { alts }
```

where `of` is, as usual, a layout herald.

The existing `do`-notation desugar is augmented to handle this as follows:

```
do { case pat <- exp of { alts }; stmts } =
  exp >>= \case { pat -> do { stmts }; alts }
```

## Examples

We now can rewrite the motivation's example as:

```hs
do
  case Good0... <- action0 of
    Bad0_0... -> ...
    Bad0_1... -> ...
  case Good1... <- action1 of
    Bad1_0... -> ...
    Bad1_1... -> ...
  ...
```

This resolved both the growing indentation and the separation issue.

Here is an example from the GHC codebase, [`loadInterface`](https://gitlab.haskell.org/ghc/ghc/-/blob/master/compiler/GHC/Iface/Load.hs#), which uses explicit layout together with `-XNonDecreasingIndentation`:

```hs
do { ...
   ; read_result <- case (wantHiBootFile home_unit eps mod from) of
                      Failed err             -> return (Failed err)
                      Succeeded hi_boot_file -> do
                        hsc_env <- getTopEnv
                        liftIO $ computeInterface hsc_env doc_str hi_boot_file mod
   ; case read_result of {
       Failed err -> do
        { let fake_iface = emptyFullModIface mod
           ; updateEps_ $ \eps ->
                   eps { eps_PIT = extendModuleEnv (eps_PIT eps) (mi_module fake_iface) fake_iface }
                   -- Not found, so add an empty iface to
                   -- the EPS map so that we don't look again
           ; return (Failed err) } ;

       Succeeded (iface, loc) ->
   let
       loc_doc = text loc
   in
   initIfaceLcl (mi_semantic_module iface) loc_doc (mi_boot iface) $
   ...
```

Using case bind syntax, we could do without explicit layout and `-XNonDecreasingIndentation` and the code would become:

```hs
do { ...
    case Succeeded hi_boot_file <- pure (wantHiBootFile home_unit eps mod from) of
        Failed err -> return (Failed err)

    hsc_env <- getTopEnv
    case Succeeded (iface, loc) <- liftIO $ computeInterface hsc_env doc_str hi_boot_file mod of
        Failed err -> do
        { let fake_iface = emptyFullModIface mod
           ; updateEps_ $ \eps ->
                   eps { eps_PIT = extendModuleEnv (eps_PIT eps) (mi_module fake_iface) fake_iface }
                   -- Not found, so add an empty iface to
                   -- the EPS map so that we don't look again
           ; return (Failed err) } ;
    let loc_doc = text loc
    initIfaceLcl (mi_semantic_module iface) loc_doc (mi_boot iface) $
    ...
```

It's much clearer to see what is the main string of computations and what is just error handling.

## Effect and Interactions

Exhaustiveness checking is very easy. Unlike the current `fail` desugar, we don't rely on any intentional incomplete patterns.


## Costs and Drawbacks

As we propose additional syntactic sugar, this adds to the amount of syntax Haskell beginners have to learn. However, this concept however already exists (varying in its syntax) in different languages, for example:

- Agda has a [pattern-bind sugar](https://agda.readthedocs.io/en/v2.5.4.1/language/syntactic-sugar.html):

    ```
    do (Good res) <- exp
        where err1 -> ret1
    cont
    ```
- Idris has a [pattern matching bind sugar](http://docs.idris-lang.org/en/latest/tutorial/interfaces.html#pattern-matching-bind):

    ```
    do (Good res) <- exp | err1 => ret1
       cont
    ```

This will make it easier for people with knowledge of these languages to get used to the syntax.

## Alternatives

### Syntactic Alternatives

[There](https://github.com/ghc-proposals/ghc-proposals/pull/327) have been proposed some slight variations to the syntax `case pat <- exp of { alts }`, namely:

1. `pat <-case exp of { alts }`
2. `pat case <- exp of { alts }`
3. `pat <- exp *keyword* { alts }`, where `keyword` is one of `of`, `where`, `else` or `else of`.

Number 1 has the disadvantage that it seems like the space between `<-` and `case` is missing accidentally, not on purpose, and adding the space changes the semantics of the program.

Number 3 has the advantage that the trailing `else { alts }` (resp. `of`, `where` or `else of`) is optional: `pat <- exp` already constitutes a valid (must-succeed) pattern match expression, and adding `else { alts }` adds failure handling to this expression.

### Non-monadic case bind

This proposal focuses only on providing syntactic sugar for pattern matches with a monadic scrutinee which are inside a do block.

The syntactic sugar for a "pattern match with default branch" could however also be proposed without the need for a monadic scrutinee as follows:

```hs
stmt → case pat = exp of { alts } in rhs
```

This would then desugar to `case exp of { pat -> rhs; alts }`.

This is however not a goal of the current proposal: we mainly see the problems which were described in the motivation (growing indentation and separation) happening in imperative-style `do`-code. We do not see a great need to mitigate these same problems for pattern matches in "normal" code.

### More

You could use the language extension `-XNondecreasingIndentation` to improve the indentation situation. This would still not solve the separation problem, however.

## Unresolved Questions

Which of the multiple syntactic alternatives is the way to go?

## Implementation Plan

The implementation will be done by @knothed.

## Endorsements

Not any so far.
