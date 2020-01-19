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

This chapter will focus on the former use case (promoting reusability) by splitting up a configuration file to follow an opinionated project layout.

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
          , actions = Actions::{
            , backport = Some Backport::{ branches = Some [ "${version}.x" ] }
            , label = Some Label::{ remove = Some [ "backport-${version}" ] }
            }
          , conditions = [ "merged", "label=backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

in  { pull_request_rules =
      [ Rule::{
        , actions = Actions::{
          , merge = Some Merge::{
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
          , actions = schemas.Actions::{
            , backport = Some schemas.Backport::{
              , branches = Some [ "${version}.x" ]
              }
            , label = Some schemas.Label::{
              , remove = Some [ "backport-${version}" ]
              }
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
        , actions = mergify.Actions::{
          , merge = Some mergify.Merge::{
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

This level of granularity is appropriate for a small project and future chapters will continue to use this smaller layout.  However, for completeness this chapter will also cover a more granular file organization suitable for larger projects.

## Large-scale project organization

As Dhall projects grow the main change is to split up top level `./types.dhall`, `./schemas.dhall` and `./utils.dhall` files into directories (with one file per type, schema or utility).

The directory layout for such a granular project would be:

```
$ tree example
example
├── mergify.dhall
├── package.dhall
├── schemas
│   ├── Actions.dhall
│   ├── Backport.dhall
│   ├── DeleteHeadBranch.dhall
│   ├── Label.dhall
│   ├── Merge.dhall
│   └── Rule.dhall
├── schemas.dhall
├── types
│   ├── Actions.dhall
│   ├── Backport.dhall
│   ├── Condition.dhall
│   ├── DeleteHeadBranch.dhall
│   ├── Label.dhall
│   ├── Merge.dhall
│   ├── Method.dhall
│   ├── RebaseFallback.dhall
│   ├── Rule.dhall
│   ├── Strict.dhall
│   └── StrictMethod.dhall
├── types.dhall
├── utils
│   └── backport.dhall
└── utils.dhall
```

We would still have top-level `./types.dhall`, `./schemas.dhall`, and `./utils.dhall` files, but all they would do is re-export definitions from individual files:

```haskell
-- ./types.dhall

{ Actions = ./types/Actions.dhall
, Backport = ./types/Backport.dhall
, Condition = ./types/Condition.dhall
, DeleteHeadBranch = ./types/DeleteHeadBranch.dhall
, Label = ./types/Label.dhall
, Merge = ./types/Merge.dhall
, Method = ./types/Method.dhall
, RebaseFallback = ./types/RebaseFallback.dhall
, Rule = ./types/Rule.dhall
, Strict = ./types/Strict.dhall
, StrictMethod = ./types/StrictMethod.dhall
}
```

```haskell
-- ./schemas.dhall

{ Actions = ./schemas/Actions
, Backport = ./schemas/Backport
, DeleteHeadBranch = ./schemas/DeleteHeadBranch
, Label = ./schemas/Label
, Merge = ./schemas/Merge
, Rule = ./schemas/Rule
}
```

```haskell
-- ./utils.dhall

{ backport = ./utils/backport.dhall }
```

This level of granularity would require greater overhead to author and maintain, so you should avoid dividing things in this way unless:

* The files get so large that they become unwieldy to navigate and edit

* You need to split things up in order to avoid cyclic imports

* You need to grant separate permissions or ownership to individual types/schemas/utilities

# Next steps

This chapter concludes the conversion of our original Mergify YAML configuration to an idiomatic Dhall project.  The next chapter covers how to keep the Dhall configuration file in sync with the generated YAML file.
