# How to organize a Dhall project

The Dhall language provides built-in support for splitting larger files into smaller files, similar to how conventional programming languages support splitting up code into multiple modules.

One common reason for splitting up Dhall configuration files is so that each file can ["do one thing and do it well"](https://en.wikipedia.org/wiki/Unix_philosophy#Do_One_Thing_and_Do_It_Well).

For example, these smaller files could be:

* reusable functions
* reusable types/schemas
* reusable constants, records, or other data

Another common reason for splitting up files is to accommodate parallel contributions from multiple individuals or teams.  Giving each contributor ownership of their respective file allows each file to potentially have:

* independent permissions
* independent code reviewers (e.g. [GitHub `CODEOWNERS`](https://help.github.com/en/github/creating-cloning-and-archiving-repositories/about-code-owners))
* independent repositories

This chapter will begin by focusing on the former use case (promoting reusability) by splitting up a configuration file to follow an opinionated project layout.  At the end of the chapter we'll also cover the latter use case (parallel contributions).

## Small-scale project organization

The previous chapter concluded with the following configuration file consisting primarily of "types" and "schemas":

```haskell
let Condition = Text

let Backport =
      { Type = { branches : Optional (List Text) }
      , default = { branches = None (List Text) }
      }

let DeleteHeadBranch = { Type = {}, default = {=} }

let Label =
      { Type = { add : Optional (List Text), remove : Optional (List Text) }
      , default = { add = None (List Text), remove = None (List Text) }
      }

let Method = < merge | squash | rebase >

let RebaseFallback = < merge | squash | null >

let Strict = < dumb : Bool | smart >

let StrictMethod = < merge | rebase >

let Merge =
      { Type =
          { method : Optional Method
          , rebase_fallback : Optional RebaseFallback
          , strict : Optional Strict
          , strict_method : Optional StrictMethod
          }
      , default =
          { method = None Method
          , rebase_fallback = None RebaseFallback
          , strict = None Strict
          , strict_method = None StrictMethod
          }
      }

let Actions =
      { Type =
          { backport : Optional Backport.Type
          , delete_head_branch : Optional DeleteHeadBranch.Type
          , label : Optional Label.Type
          , merge : Optional Merge.Type
          }
      , default =
          { backport = None Backport.Type
          , delete_head_branch = None DeleteHeadBranch.Type
          , label = None Label.Type
          , merge = None Merge.Type
          }
      }

let Rule =
      { Type =
          { name : Text, conditions : List Condition, actions : Actions.Type }
      , default =
          { conditions = [] : List Condition, actions = Actions.default }
      }

let backport =
          \(version : Text)
      ->  Rule::{
          , actions =
              Actions::{
              , backport = Some Backport::{ branches = Some [ "${version}.x" ] }
              , label =
                  Some Label::{ remove = Some [ "backport-${version}" ] }
              }
          , conditions = [ "merged", "label=backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

in  { pull_request_rules =
        [ Rule::{
          , actions =
              Actions::{
              , merge =
                  Some
                    Merge::{
                    , method = Some Method.squash
                    , strict = Some Strict.smart
                    }
              }
          , conditions =
              [ "status-success=continuous-integration/appveyor/pr"
              , "label=merge me"
              , "#approved-reviews-by>=1"
              ]
          , name = "Automatically merge pull requests"
          }
        , Rule::{
          , actions = Actions::{ delete_head_branch = Some {=} }
          , conditions = [ "merged" ]
          , name = "Delete head branch after merge"
          }
        , backport "1.0"
        , backport "1.1"
        , backport "1.2"
        , backport "1.3"
        , backport "1.4"
        , backport "1.5"
        ]
    }
```

Here we use "type" to refer to an ordinary Dhall type, like `Condition`:

```haskell
let Condition = Text

...
```

... and we use "schema" to refer to a record containing a `Type` and a `default` field, suitable for use with the `::` operator.  For example, `Backport` is a "schema":

```haskell
let Backport =
      { Type = { branches : Optional (List Text) }
      , default = { branches = None (List Text) }
      }

...
```

We'll use this same naming convention for our project layout by creating a
`./types.dhall` file:

```haskell
-- ./types.dhall

let Condition = Text

let Backport = { branches : Optional (List Text) }

let DeleteHeadBranch = {}

let Label = { add : Optional (List Text), remove : Optional (List Text) }

let Method = < merge | squash | rebase >

let RebaseFallback = < merge | squash | null >

let Strict = < dumb : Bool | smart >

let StrictMethod = < merge | rebase >

let Merge =
      { method : Optional Method
      , rebase_fallback : Optional RebaseFallback
      , strict : Optional Strict
      , strict_method : Optional StrictMethod
      }

let Actions =
      { backport : Optional Backport
      , delete_head_branch : Optional DeleteHeadBranch
      , label : Optional Label
      , merge : Optional Merge
      }

let Rule = { name : Text, conditions : List Condition, actions : Actions }

in  { Condition = Condition
    , Backport = Backport
    , DeleteHeadBranch = DeleteHeadBranch
    , Label = Label
    , Method = Method
    , RebaseFallback = RebaseFallback
    , Strict = Strict
    , StrictMethod = StrictMethod
    , Merge = Merge
    , Actions = Actions
    , Rule = Rule
    }
```

... and a `./schemas.dhall` file:

```haskell
-- ./schemas.dhall

let types = ./types.dhall

let Backport =
      { Type = types.Backport, default = { branches = None (List Text) } }

let DeleteHeadBranch = { Type = types.DeleteHeadBranch, default = {=} }

let Label =
      { Type = types.Label
      , default = { add = None (List Text), remove = None (List Text) }
      }

let Merge =
      { Type = types.Merge
      , default =
          { method = None types.Method
          , rebase_fallback = None types.RebaseFallback
          , strict = None types.Strict
          , strict_method = None types.StrictMethod
          }
      }

let Actions =
      { Type = types.Actions
      , default =
          { backport = None types.Backport
          , delete_head_branch = None types.DeleteHeadBranch
          , label = None types.Label
          , merge = None types.Merge
          }
      }

let Rule =
      { Type = types.Rule
      , default =
          { conditions = [] : List types.Condition, actions = Actions.default }
      }

in  { Backport = Backport
    , DeleteHeadBranch = DeleteHeadBranch
    , Label = Label
    , Merge = Merge
    , Actions = Actions
    , Rule = Rule
    }
```

We can also create a `./utils.dhall` file containing our `backport` function:

```haskell
-- ./utils.dhall

let types = ./types.dhall

let schemas = ./schemas.dhall

let backport
    : Text -> types.Rule
    =     \(version : Text)
      ->  schemas.Rule::{
          , actions =
              schemas.Actions::{
              , backport =
                  Some schemas.Backport::{ branches = Some [ "${version}.x" ] }
              , label =
                  Some
                    schemas.Label::{ remove = Some [ "backport-${version}" ] }
              }
          , conditions = [ "merged", "label=backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

in  { backport = backport }
```

For convenience, we can also create a `package.dhall` file which combines them all:

```haskell
-- ./package.dhall

let types = ./types.dhall

let schemas = ./schemas.dhall

let utils = ./utils.dhall

in  types // schemas // utils
```

Carefully note that we use the `//` operator to prefer the schema if there is both a type and schema of the same name (e.g.  `types.Action` and `schemas.Action`).  We prefer the schema since the corresponding type can still be accessed as a field of the schema (e.g. `schemas.Action.Type`).

Now we can use `package.dhall` to simplify our original Mergify configuration file:

```haskell
let mergify = ./package.dhall

in  { pull_request_rules =
        [ mergify.Rule::{
          , actions =
              mergify.Actions::{
              , merge =
                  Some
                    mergify.Merge::{
                    , method = Some mergify.Method.squash
                    , strict = Some mergify.Strict.smart
                    }
              }
          , conditions =
              [ "status-success=continuous-integration/appveyor/pr"
              , "label=merge me"
              , "#approved-reviews-by>=1"
              ]
          , name = "Automatically merge pull requests"
          }
        , mergify.Rule::{
          , actions = mergify.Actions::{ delete_head_branch = Some {=} }
          , conditions = [ "merged" ]
          , name = "Delete head branch after merge"
          }
        , mergify.backport "1.0"
        , mergify.backport "1.1"
        , mergify.backport "1.2"
        , mergify.backport "1.3"
        , mergify.backport "1.4"
        , mergify.backport "1.5"
        ]
    }
```

Here we name the imported package after the project (e.g. `mergify`), which is an idiom that comes in handy when you start importing more than one package.

This is level of granularity is appropriate for a small project of this size and future chapters will reuse this smaller layout.  However, for completeness this chapter will also cover a more granular file organization suitable for larger projects.

## Large-scale project organization

As Dhall projects grow the main change is to split up top level `./types.dhall`, `./schemas.dhall` and `./utils.dhall` files into directories (with one file per type, schema or utility).

For example, we can split up the `./types.dhall` file into the following smaller files underneath a `./types/` subdirectory:

```haskell
-- ./types/Condition.dhall

{ branches : Optional (List Text) }
```

```haskell
-- ./types/Backport.dhall

{ branches : Optional (List Text) }
```

```haskell
-- ./types/DeleteHeadBranch.dhall

{}
```

```haskell
-- ./types/Label.dhall

{ add : Optional (List Text), remove : Optional (List Text) }
```

```haskell
-- ./types/Method.dhall

< merge | squash | rebase >
```

```haskell
-- ./types/RebaseFallback.dhall

< merge | squash | null >
```

```haskell
-- ./types/Strict.dhall

< dumb : Bool | smart >
```

```haskell
-- ./types/StrictMethod.dhall

< merge | rebase >
```

```haskell
-- ./types/Merge.dhall

let Method = ./Method.dhall

let RebaseFallback = ./RebaseFallback.dhall

let Strict = ./Strict.dhall

let StrictMethod = ./StrictMethod.dhall

in  { method : Optional Method
    , rebase_fallback : Optional RebaseFallback
    , strict : Optional Strict
    , strict_method : Optional StrictMethod
    }
```

```haskell
-- ./types/Actions.dhall

let Backport = ./Backport.dhall

let DeleteHeadBranch = ./DeleteHeadBranch.dhall

let Label = ./Label.dhall

let Merge = ./Merge.dhall

in  { backport : Optional Backport
    , delete_head_branch : Optional DeleteHeadBranch
    , label : Optional Label
    , merge : Optional Merge
    }
```

```haskell
-- ./types/Rule.dhall

let Condition = ./Condition.dhall

let Actions = ./Actions.dhall

in  { name : Text, conditions : List Condition, actions : Actions }
```

... and similarly split up `./schemas.dhall` into files underneath a `./schemas/` subdirectory:

```haskell
-- ./schemas/Backport.dhall

let types = ../types.dhall

in  { Type = types.Backport, default = { branches = None (List Text) } }
```

```haskell
-- ./schemas/DeleteHeadBranch.dhall

let types = ../types.dhall in { Type = types.DeleteHeadBranch, default = {=} }
```

```haskell
-- ./schemas/Label.dhall

let types = ../types.dhall

in  { Type = types.Label
    , default = { add = None (List Text), remove = None (List Text) }
    }
```

```haskell
-- ./schemas/Merge.dhall

let types = ../types.dhall

in  { Type = types.Merge
    , default =
        { method = None types.Method
        , rebase_fallback = None types.RebaseFallback
        , strict = None types.Strict
        , strict_method = None types.StrictMethod
        }
    }
```

```haskell
-- ./schemas/Actions.dhall

let types = ../types.dhall

in  { Type = types.Actions
    , default =
        { backport = None types.Backport
        , delete_head_branch = None types.DeleteHeadBranch
        , label = None types.Label
        , merge = None types.Merge
        }
    }
```

```haskell
-- ./schemas/Rule.dhall

let types = ../types.dhall

let Actions = ./Actions.dhall

in  { Type = types.Rule
    , default = { conditions = [] : List types.Condition, actions = Actions }
    }
```

... and split up `./utils.dhall`:

```haskell
-- ./utils/backport.dhall

let types = ../types.dhall

let schemas = ../schemas.dhall

let backport
    : Text -> types.Rule
    =     \(version : Text)
      ->  schemas.Rule::{
          , actions =
              schemas.Actions::{
              , backport =
                  Some schemas.Backport::{ branches = Some [ "${version}.x" ] }
              , label =
                  Some
                    schemas.Label::{ remove = Some [ "backport-${version}" ] }
              }
          , conditions = [ "merged", "label=backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

in  backport
```

## `let`-bound imports

Note that the above files consistently import their dependencies using a separate `let` expression for each dependency, like this:

```haskell
-- ./types/Actions.dhall

let Backport = ./Backport.dhall

let DeleteHeadBranch = ./DeleteHeadBranch.dhall

let Label = ./Label.dhall

let Merge = ./Merge.dhall

in  { backport : Optional Backport
    , delete_head_branch : Optional DeleteHeadBranch
    , label : Optional Label
    , merge : Optional Merge
    }
```

This "`let`-bound import" convention is not required by the language.  Inline imports work and we could have instead written `./Actions.dhall` like this:

```haskell
-- ./types/Actions.dhall

{ backport : Optional ./Backport.dhall
, delete_head_branch : Optional ./DeleteHeadBranch.dhall
, label : Optional ./Label.dhall
, merge : Optional ./Merge.dhall
}
```

The primary reasons for `let`-bound imports are:

* `let`-bound imports are more refactor-friendly

  For example, suppose we change the `./types/Label.dhall` file to reference a new
  `./LabelName.dhall` file:

  ```haskell
  -- ./types/LabelName.dhall

  Text
  ```

  ```haskell
  -- ./types/Label.dhall

  let LabelName = ./LabelName.dhall

  in  { add : Optional (List LabelName), remove : Optional (List LabelName) }
  ```

  If we were to relocate `./types/LabelName.dhall` to `./types/Label/LabelName.dhall` we would only need to fix one occurrence of the import because we `let`-bound the import.

* `let`-bound imports avoid cluttering the code with integrity checks

  Later on we'll cover how optionally to protect imports using semantic integrity checks, like this:

  ```haskell
  let Backport =
        ./Backport.dhall sha256:2578e6326c7f8f5750206a5fce98527b16baf977bc98fc95a26a33ba850829cd

  let DeleteHeadBranch =
        ./DeleteHeadBranch.dhall sha256:0912602a19e01dcff30f351958d2d9b69519c9be61b57b1b32a2a569bf8bf5f9

  let Label =
        ./Label.dhall sha256:dccb5abc63f3af74c02058dc5db86cc036a97839b0862180a1b3379958826d9e

  let Merge =
        ./Merge.dhall sha256:d3c0658281f67aebe83ceb3bc6d80f9f680d8147e9a03934110bf4b19a17234b

  in  { backport : Optional Backport
      , delete_head_branch : Optional DeleteHeadBranch
      , label : Optional Label
      , merge : Optional Merge
      }
  ```

  ... and we would like to avoid integrity checks polluting the the main expression we care about, like this:

  ```haskell
  { backport :
      Optional
        ./Backport.dhall sha256:2578e6326c7f8f5750206a5fce98527b16baf977bc98fc95a26a33ba850829cd
  , delete_head_branch :
      Optional
        ./DeleteHeadBranch.dhall sha256:0912602a19e01dcff30f351958d2d9b69519c9be61b57b1b32a2a569bf8bf5f9
  , label :
      Optional
        ./Label.dhall sha256:dccb5abc63f3af74c02058dc5db86cc036a97839b0862180a1b3379958826d9e
  , merge :
      Optional
        ./Merge.dhall sha256:d3c0658281f67aebe83ceb3bc6d80f9f680d8147e9a03934110bf4b19a17234b
  }
  ```

* `let`-bound imports also avoid cluttering the code with long relative paths

  As your project grows you might begin to switch from a flat directory layout to a nested one where relative import paths grow longer.  Inlining these large relative paths pollutes the body of the code just like inlining integrity checks.

* `let`-bound imports will be more familiar to most programmers

  Many other programming languages require program modules to begin with an import list, so following this convention within Dhall helps newcomers recognize that they are reading an import list.

Consequently, this book will continue to use the convention of `let`-bound imports.

## Re-exports

We'll still have top-level `./types.dhall`, `./schemas.dhall`, and `./utils.dhall` files, but all they will do is re-export definitions from individual files:

```haskell
-- ./types.dhall

let Actions = ./types/Actions.dhall

let Backport = ./types/Backport.dhall

let Condition = ./types/Condition.dhall

let DeleteHeadBranch = ./types/DeleteHeadBranch.dhall

let Label = ./types/Label.dhall

let Merge = ./types/Merge.dhall

let Method = ./types/Method.dhall

let RebaseFallback = ./types/RebaseFallback.dhall

let Rule = ./types/Rule.dhall

let Strict = ./types/Strict.dhall

let StrictMethod = ./types/StrictMethod.dhall

in  { Actions = Actions
    , Backport = Backport
    , Condition = Condition
    , DeleteHeadBranch = DeleteHeadBranch
    , Label = Label
    , Merge = Merge
    , Method = Method
    , RebaseFallback = RebaseFallback
    , Rule = Rule
    , Strict = Strict
    , StrictMethod = StrictMethod
    }
```

```haskell
-- ./schemas.dhall

let Actions = ./schemas/Actions.dhall

let Backport = ./schemas/Backport.dhall

let DeleteHeadBranch = ./schemas/DeleteHeadBranch.dhall

let Label = ./schemas/Label.dhall

let Merge = ./schemas/Merge.dhall

let Rule = ./schemas/Rule.dhall

in  { Actions = Actions
    , Backport = Backport
    , DeleteHeadBranch = DeleteHeadBranch
    , Label = Label
    , Merge = Merge
    , Rule = Rule
    }
```

```haskell
-- ./utils.dhall

let backport = ./utils/backport.dhall

in  { backport = backport }
```

## Parallel contributions

Suppose that we want to split up our configuration file into two separate configuration files:

* One file for the backport-related rules

* One file for everything else

Here there is no convention for how to organize or name things (other than perhaps to not pollute the top-level directory of your project).  Since this is a Mergify configuration we can create a `./rules/` subdirectory to store files containing Mergify rules and underneath that directory we'll create two files:

```haskell
-- ./rules/merge.dhall

let schemas = ../package.dhall

in  [ schemas.Rule::{
      , actions =
          schemas.Actions::{
          , merge =
              Some
                schemas.Merge::{
                , method = Some schemas.Method.squash
                , strict = Some schemas.Strict.smart
                }
          }
      , conditions =
          [ "status-success=continuous-integration/appveyor/pr"
          , "label=merge me"
          , "#approved-reviews-by>=1"
          ]
      , name = "Automatically merge pull requests"
      }
    , schemas.Rule::{
      , actions = schemas.Actions::{ delete_head_branch = Some {=} }
      , conditions = [ "merged" ]
      , name = "Delete head branch after merge"
      }
    ]
```

```haskell
-- ./rules/backport.dhall

let utils = ../utils.dhall

in  [ utils.backport "1.0"
    , utils.backport "1.1"
    , utils.backport "1.2"
    , utils.backport "1.3"
    , utils.backport "1.4"
    , utils.backport "1.5"
    ]
```

... which we can then reference from our top-level configuration:

```haskell
-- ./mergify.dhall

let mergeRules = ./rules/merge.dhall

let backportRules = ./rules/backport.dhall

in  { pull_request_rules = mergeRules # backportRules }
```

## Next steps

Now that we've split our project configuration into two separate rules files we can begin automating the process of adding new backport rules.

The next chapter will cover the use of the Prelude package, which provides several generic utilities to automate common tasks like this one.
