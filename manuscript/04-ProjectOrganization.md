# How to organize a Dhall project

The Dhall language provides built-in support for splitting large configuration files into smaller files, like other programming languages.

One common reason for splitting up files is so that each file can ["do one thing and do it well"](https://en.wikipedia.org/wiki/Unix_philosophy#Do_One_Thing_and_Do_It_Well).

For example, these smaller individual files could be:

* reusable functions
* reusable types/schemas
* reusable constants, records, or other data

Another common reason for splitting up files is to accommodate multiple individuals or teams to contribute in parallel to a shared overarching configuration.  Giving each contributor ownership of their respective file allows each file to have:

* independent permissions
* independent code reviewers (e.g. [GitHub `CODEOWNERS`](https://help.github.com/en/github/creating-cloning-and-archiving-repositories/about-code-owners))
* independent repositories

This chapter will begin by focusing on the former use case (promoting reusability) by splitting up the following configuration file taken from the previous chapter to follow an opinionated Dhall project layout.  At the end of the chapter we'll also cover the latter use case (parallel contributions).

## Types

The last chapter concluded with the following configuration file consisting primarily of types and schemas:

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

We'll use this same naming convention for our project layout by creating two subdirectories named `./types/` and `./schemas/`.

Let's begin by creating the following files underneath the `./types/` directory (except without the initial comment line):

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

Dhall expressions (such as types) can import other Dhall expressions (such as other nested types) by referencing their relative or absolute path.  Internal imports within a project should use relative paths so that the project can be safely relocated without breaking imports.

Currently we are using a flat hierarchy for our directory of types, meaning that all of our types live underneath a single non-nested `./types/` directory.  For a more complex project we might want to create a project-specific organization for your types.

For example, all of the actions (like `Backport`, `DeleteHeadBranch`, `Label`, and `Merge`) could live underneath a `./types/Actions/` subdirectory.  However, for this chapter we'll stick to a flat layout to keep things short.

## `let`-bound imports

Note that the above types import their dependencies using a separate `let` expression for each dependency, even though this is not required.  The Dhall language does support inline imports and we could have written `./Actions.dhall` like this:

```haskell
{ backport : Optional ./Backport.dhall
, delete_head_branch : Optional ./DeleteHeadBranch.dhall
, label : Optional ./Label.dhall
, merge : Optional ./Merge.dhall
}
```

The primary reasons for the `let`-bound import convention are:

* `let`-bound imports are more refactor-friendly

  For example, suppose we change `./types/Label.dhall` to reference a new
  `./LabelName.dhall`:

  ```haskell
  -- ./types/LabelName.dhall
  Text
  ```

  ```haskell
  -- ./types/Label.dhall
  let LabelName = ./LabelName.dhall

  in  { add : Optional (List LabelName), remove : Optional (List LabelName) }
  ```

  Now if we were to relocate `./types/LabelName.dhall` we only need to fix one occurrence of the import instead of two occurrences because we import `LabelName` once and reuse the imported value.

* `let`-bound imports avoid cluttering the code with integrity checks

  Later on we'll cover how to protect imports using semantic integrity checks, like this:

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

  ... we want to avoid integrity checks from polluting the body of the main expression we care about, like this:

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

* `let`-bound imports avoid cluttering the code with long relative paths

  As a project grows you begin to switch from a flat directory layout to a nested one and relative import paths begin to grow longer.  Inlining these large relative paths pollutes the body of the code just like inlining integrity checks.

* `let`-bound imports will be more familiar to most programmers

  Many other programming languages require program modules to begin with an import list, so this convention helps newcomers to Dhall recognize that they are reading an import list.

So this book will continue to use the convention of `let`-bound imports, even for small files.

## Creating a package file

For ease of use, we can create a `./types.dhall` file in the root of our project that packages all of our types into a record, for convenience:

```haskell
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

There are two reasons for creating this package file:

* To minimize imports for downstream users of our types

  Now they can import a single types record and access types as fields of that file instead of importing each type individually

* To make our project more refactor-friendly

  If we decide to reorganize our `./types/` directory from a flat to a nested directory layout then all we need to do is update our `./types.dhall` file and then downstream code that imports `./types.dhall` remains unaffected.

## Schemas

Now that we've created our `./types.dhall` package we can create a `./schemas` directory:

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

... and a `./schemas.dhall` file:

```haskell
{ Actions = ./schemas/Actions.dhall
, Backport = ./schemas/Backport.dhall
, DeleteHeadBranch = ./schemas/DeleteHeadBranch.dhall
, Label = ./schemas/Label.dhall
, Merge = ./schemas/Merge.dhall
, Rule = ./schemas/Rule.dhall
}
```

## Using our package files

Now we can reference `./types.dhall` and `./schemas.dhall` to simplify our original Mergify configuration, like this:

```haskell
let types = ./types.dhall

let schemas = ./schemas.dhall

let backport =
          \(version : Text)
      ->  schemas.Rule::{
          , actions =
              schemas.Actions::{
              , backport =
                  Some schemas.Backport::{ branches = Some [ "${version}.x" ] }
              , label =
                  Some
                    schemas.Label::{
                    , remove = Some [ "backport-${version}" ]
                    }
              }
          , conditions = [ "merged", "label=backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

in  { pull_request_rules =
        [ schemas.Rule::{
          , actions =
              schemas.Actions::{
              , merge =
                  Some
                    schemas.Merge::{
                    , method = Some types.Method.squash
                    , strict = Some types.Strict.smart
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
        , backport "1.0"
        , backport "1.1"
        , backport "1.2"
        , backport "1.3"
        , backport "1.4"
        , backport "1.5"
        ]
    }
```

We can even go a step further and export both schemas and types from a top-level `./package.dhall` file, like this:

```haskell
-- ./package.dhall
let types = ./types.dhall

let schemas = ./schemas.dhall

in  types // schemas
```

Carefully note that if there is both a type and schema of the same name (e.g.  `types.Action` and `schemas.Action`), we prefer to package the schema since the type can be accessed as a field of the schema (e.g. `schemas.Action.Type`).

Then we can use that single import for our configuration file:

```haskell
let mergify = ./package.dhall

let backport =
          \(version : Text)
      ->  mergify.Rule::{
          , actions =
              mergify.Actions::{
              , backport =
                  Some mergify.Backport::{ branches = Some [ "${version}.x" ] }
              , label =
                  Some
                    mergify.Label::{
                    , remove = Some [ "backport-${version}" ]
                    }
              }
          , conditions = [ "merged", "label=backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

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
        , backport "1.0"
        , backport "1.1"
        , backport "1.2"
        , backport "1.3"
        , backport "1.4"
        , backport "1.5"
        ]
    }
```

Here we now name the imported package after the project (e.g. `mergify`), which will come in handy later on when we want to import more than one package.

## Parallel contributions

Suppose that we want to split up our configuration file into two separate configuration files:

* One file for the backport-related rules

  Perhaps because we want backport rules to be auto-generated by a script with each new release

* One file for everything else

Here there is no convention for how to organize or name things (other than perhaps to not pollute the top-level directory of your project).  Since this is a Mergify configuration we can create a `./rules/` subdirectory to store files containing Mergify rules.

Underneath that directory we'll create two files:

```haskell
-- ./rules/merge.dhall
let mergify = ../package.dhall

in  [ mergify.Rule::{
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
    ]
```

```haskell
-- ./rules/backport.dhall

let mergify = ../package.dhall

let backport =
          \(version : Text)
      ->  mergify.Rule::{
          , actions =
              mergify.Actions::{
              , backport =
                  Some mergify.Backport::{ branches = Some [ "${version}.x" ] }
              , label =
                  Some
                    mergify.Label::{
                    , remove = Some [ "backport-${version}" ]
                    }
              }
          , conditions = [ "merged", "label=backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

in  [ backport "1.0"
    , backport "1.1"
    , backport "1.2"
    , backport "1.3"
    , backport "1.4"
    , backport "1.5"
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
