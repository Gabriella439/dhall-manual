# How to support default values

This chapter describes the common idiom of defining a record of default values, using the Mergify configuration file from the end of the previous chapter as an illustrative example.

## The Problem

Most configuration files let you omit options to specify that you prefer their default values.

For example, suppose we want to select the default merge method:

```yaml
pull_request_rules:

  - name: Automatically merge pull requests
    conditions:
      - status-success=continuous-integration/appveyor/pr
      - label=merge me
      - ! '#approved-reviews-by>=1'
    actions:
      merge:
        strict: smart
        method: squash  # We want to change this to the default value
```

We would need to delete the field in order to specify that we prefer the default, like this:

```yaml
pull_request_rules:

  - name: Automatically merge pull requests
    conditions:
      - status-success=continuous-integration/appveyor/pr
      - label=merge me
      - ! '#approved-reviews-by>=1'
    actions:
      merge:
        strict: smart
        # No more `merge` field
```

Configuration files should preserve the user's intent and if we intend to select the default then we should omit the field.  Omission ensures that we continue to track changes to the default value.

Unfortunately, we cannot add or remove fields from Dhall records without changing their type, so Dhall needs a way to model the presence or absence of a field.

## `Optional` fields

You can model default-valued fields as `Optional` fields of a record.  For example, in the previous chapter we ended on this representation for merge-related options:

```haskell
let Method = < merge | rebase | squash >

let RebaseFallback = < merge | null | squash >

let Strict = < dumb : Bool | smart >

let StrictMethod = < merge | rebase >

in  { method :
        Optional Method
    , rebase_fallback :
        Optional RebaseFallback
    , strict :
        Optional Strict
    , strict_method :
        Optional StrictMethod
    }
```

... which means that this YAML record:

```yaml
        strict: smart
        method: squash
```

... corresponds to this Dhall record when constrained to that type:

```haskell
let Method = < merge | rebase | squash >

let RebaseFallback = < merge | null | squash >

let Strict = < dumb : Bool | smart >

let StrictMethod = < merge | rebase >

in  { method =
        Some Method.squash
    , rebase_fallback :
        None RebaseFallback
    , strict :
        Some Strict.smart
    , strict_method :
        None StrictMethod
    }
```

However, this becomes cumbersome when we have a record with a large number of `Optional` fields, most of which are `None`.  We'd like to refactor our code so that we only need to specify non-default fields (just like the original YAML configuration).

## Records of defaults

We can consolidate all of the `None`s in one place by defining a record of defaults where all of the non-required fields are set to `None`.

For example, we can take the Dhall configuration file from the end of the
previous chapter:

```dhall
let Condition = Text

let Backport = { branches : Optional (List Text) }

let DeleteHeadBranch = {}

let Label = { add : Optional (List Text), remove : Optional (List Text) }

let Method = < merge | squash | rebase >

let RebaseFallback = < merge | squash | null >

let Strict = < dumb : Bool | smart >

let StrictMethod = < merge | rebase >

let Merge =
      { method :
          Optional Method
      , rebase_fallback :
          Optional RebaseFallback
      , strict :
          Optional Strict
      , strict_method :
          Optional StrictMethod
      }

let Actions =
      { backport :
          Optional Backport
      , delete_head_branch :
          Optional DeleteHeadBranch
      , label :
          Optional Label
      , merge :
          Optional Merge
      }

let Rule = { name : Text, conditions : List Condition, actions : Actions }

let backport =
        \(version : Text)
      → { actions =
            { backport =
                Some { branches = Some [ "${version}.x" ] }
            , delete_head_branch =
                None DeleteHeadBranch
            , label =
                Some
                { add =
                    None (List Text)
                , remove =
                    Some [ "status:backport-${version}" ]
                }
            , merge =
                None Merge
            }
        , conditions =
            [ "merged", "label=status:backport-${version}" ]
        , name =
            "backport patches to ${version}.x branch"
        }

in  { pull_request_rules =
        [ { actions =
              { backport =
                  None Backport
              , delete_head_branch =
                  None DeleteHeadBranch
              , label =
                  None Label
              , merge =
                  Some
                  { method =
                      Some < merge | rebase | squash >.squash
                  , rebase_fallback =
                      None RebaseFallback
                  , strict =
                      Some < dumb : Bool | smart >.smart
                  , strict_method =
                      None StrictMethod
                  }
              }
          , conditions =
              [ "status-success=continuous-integration/appveyor/pr"
              , "label=merge me"
              , "#approved-reviews-by>=1"
              ]
          , name =
              "Automatically merge pull requests"
          }
        , { actions =
              { backport =
                  None Backport
              , delete_head_branch =
                  Some {=}
              , label =
                  None Label
              , merge =
                  None Merge
              }
          , conditions =
              [ "merged" ]
          , name =
              "Delete head branch after merge"
          }
        , backport "1.0"
        , backport "1.1"
        , backport "1.2"
        , backport "1.3"
        , backport "1.4"
        ]
    }
```

... and define a record of defaults for each record type with default-valued
fields:
