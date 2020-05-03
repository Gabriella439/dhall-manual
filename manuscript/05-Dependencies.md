# How to install and manage Dhall packages

Many Dhall projects depend on the Dhall Prelude for commonly used utilities or integration-specific packages (such as `dhall-kubernetes`) for higher-level utilities to simplify their workflow.  This chapter explains how to install and use packages like the Prelude and maintain these dependencies over time.

We'll motivate the use of the Prelude by eliminating a small bit of repetition at the end of our running configuration file:

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

Ideally we would only need to specify the latest minor version and then we could generate `backport` functions for all releases up until the latest release, like this:

```haskell
...

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
      ] # mergify.backports { majorVersion = 1, latestMinorVersion = 5 }
    }
```

In fact, if we simplified things in this way we could factor out the latest minor version number into a separate file:

```haskell
      ...
      ] # mergify.backports { majorVersion = 1, latestMinorVersion = ./latestMinorVersion.dhall }
      ...
```

... and then a script could increment the number stored within `./latestMinorVersion.dhall` on every release boundary.

## The Prelude

We'll implement the `mergify.backports` function using a new `List/generate` utility packaged with the Prelude.  We'll begin from the final implementation and work backwards from there:

```dhall
-- ./utils.dhall

let Prelude = https://prelude.dhall-lang.org/package.dhall

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

let backports
    : { majorVersion : Natural, minorVersion : Natural } -> List types.Rule
    = \(args : { majorVersion : Natural, minorVersion : Natural }) ->
        Prelude.List.generate
          (args.minorVersion + 1)
          types.Rule
          ( \(minor : Natural) ->
              backport "${Natural/show args.majorVersion}.${Natural/show minor}"
          )

in  { backport, backports }
```

... and we can use this new `backports` function within our top-level configuration like this:

```dhall
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
      ] # mergify.backports { majorVersion = 1, minorVersion = 5 }
    }
```

The first line of our `./utils.dhall` file illustrates how we import remote packages:

```dhall
let Prelude = https://prelude.dhall-lang.org/v15.0.0/package.dhall
```

We use essentially the same idiom as importing local packages, except that instead of providing a file path we specify a URL.  Like file paths, URLs can potentially contain arbitrary Dhall expressions (e.g. functions, lists, strings, etc.).  However, the vast majority of packages are records (just like the local `./package.dhall` expression we created in the previous chapter).

The Prelude is a nested record, which we can explore using the `dhall repl`,
specifically the REPL's support for tab completion:

```dhall
$ dhall repl
⊢ :let Prelude = https://prelude.dhall-lang.org/v15.0.0/package.dhall
…

⊢ Prelude.<TAB>
Prelude.Bool      Prelude.JSON      Prelude.Monoid    Prelude.XML
Prelude.Double    Prelude.List      Prelude.Natural
Prelude.Function  Prelude.Location  Prelude.Optional
Prelude.Integer   Prelude.Map       Prelude.Text
⊢ Prelude.List.<TAB>
Prelude.List.all        Prelude.List.filter     Prelude.List.map
Prelude.List.any        Prelude.List.fold       Prelude.List.null
Prelude.List.build      Prelude.List.generate   Prelude.List.partition
Prelude.List.concat     Prelude.List.head       Prelude.List.replicate
Prelude.List.concatMap  Prelude.List.indexed    Prelude.List.reverse
Prelude.List.default    Prelude.List.iterate    Prelude.List.shifted
Prelude.List.drop       Prelude.List.last       Prelude.List.take
Prelude.List.empty      Prelude.List.length     Prelude.List.unzip
```

In this case our function of interest is `Prelude.List.generate`, meaning
that the Prelude record has a field named `List` that is a record, and that
record in turn has a field named `generate` containing the function.

We call the function `List/generate` because you can also access that
function directly by that path instead of importing the entire Prelude record:

```dhall
⊢ :let List/generate = https://prelude.dhall-lang.org/List/generate

List/generate : ∀(n : Natural) → ∀(a : Type) → ∀(f : Natural → a) → List a
```

The function's name and type give a hint to what the function does: given a
`Natural` number (`n`) and a generating function `f`, create a `List` of `n`
values.

However, you don't have to take my word for this.  You can see for yourself
how the function works by applying the function to one argument:

```dhall
⊢ List/generate 10

  λ(a : Type)
→ λ(f : Natural → a)
→ [ f 0, f 1, f 2, f 3, f 4, f 5, f 6, f 7, f 8, f 9 ]
```

Our `backports` function uses `List/generate` to create the backport list by
applying the `backport` function to all minor versions up to and including the
latest minor version.

```dhall
let backports
    : { majorVersion : Natural, minorVersion : Natural } -> List types.Rule
    = \(args : { majorVersion : Natural, minorVersion : Natural }) ->
        Prelude.List.generate
          (args.minorVersion + 1)
          types.Rule
          ( \(minor : Natural) ->
              backport "${Natural/show args.majorVersion}.${Natural/show minor}"
          )
```

The first line of our expression imports the Prelude package by specifying (A) the URL to obtain the package and (B) an integrity check to make the package tamper-proof.

```dhall
let Prelude = https://prelude.dhall-lang.org/v15.0.0/package.dhall sha256:6b90326dc39ab738d7ed87b970ba675c496bed0194071b332840a87261649dcd
```

You don't need to know the integrity check in advance.  Instead, you can omit the integrity check, like this:

```dhall
let Prelude = https://prelude.dhall-lang.org/v15.0.0/package.dhall
```

... and then run:

```bash
$ dhall freeze --inplace ./utils.dhall
```

... to compute the desired integrity check.  Alternatively, if you don't want `dhall freeze` to re-format your code you can use:

```bash
$ dhall hash <<< 'https://prelude.dhall-lang.org/v15.0.0/package.dhall'
sha256:6b90326dc39ab738d7ed87b970ba675c496bed0194071b332840a87261649dcd
```

... and manually insert the integrity check after the import.

The integrity check is optional, but recommended, because the check provides two benefits:

* The integrity check guarantees you never download a corrupt or compromised package

  The interpreter will fail fast when encountering a package that does not match the declared integrity check.

* The interpreter will automatically "install" a local copy of any package protected in this way

  "Installing" a package caches the corresponding Dhall expression on your machine so that the package never needs to be remotely fetched again.

The interpreter attempts to install packages to one of the following directories:

* `${XDG_CACHE_HOME}/dhall`
* `${LOCALAPPDATA}/dhall`
* `~/.cache/dhall`

... in that order.  Expressions are cached in a compact binary format and the name of each expression is `1220${HASH}`.  So, for example, the above Prelude package will be saved to:

```dhall
~/.cache/dhall/12206b90326dc39ab738d7ed87b970ba675c496bed0194071b332840a87261649dcd
```

... if neither the `XDG_CACHE_HOME` nor `LOCALAPPDATA` environment variables are set.



## Prelude

You can find the Prelude package by visiting:

* [The Dhall Prelude - https://prelude.dhall-lang.org](https://prelude.dhall-lang.org/)

This package has a useful `README` which explains the idioms for obtaining files 

## Package discovery

## Next steps

This chapter concludes the conversion of our original Mergify YAML configuration to an idiomatic Dhall project.  The next chapter covers how to keep the Dhall configuration file in sync with the generated YAML file.
