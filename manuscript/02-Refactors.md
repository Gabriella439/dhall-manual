# How to safely refactor a configuration file

The Dhall language and tooling are optimized for "maintainability" and one
aspect of that is ease of refactoring existing configuration files.

This chapter begins from the repetitive Dhall configuration we generated from YAML at the end of the previous chapter and uses functions to reduce repetition.  We will also verify that the refactor is behavior-preserving along the way.

## Semantic integrity checks

Before starting our refactor we can create a digest (a.k.a. a checksum) of the result, like this:

```bash
$ dhall hash --file ./mergify.dhall 
sha256:40e5e1ea3553a14ae667e292506f70b74f02501b86cf496753f8f059fd939c2f
```

If our refactor does not change the result then the digest should remain the same.

This digest is known as a "semantic integrity check" or a "semantic hash" for short.  Unlike a traditional digest this is a hash of a CBOR encoding of the syntax tree in a canonical normal form.

In other words, you could obtain the same hash by chaining the following steps:

```bash
$ dhall --alpha --file ./mergify.dhall | dhall encode | shasum --algorithm 256 --binary
40e5e1ea3553a14ae667e292506f70b74f02501b86cf496753f8f059fd939c2f  *-
```

... where:

* `dhall --alpha` reduces an expression to a canonical normal form
* `dhall encode` encodes the expression according to a [standard CBOR representation](https://github.com/dhall-lang/dhall-lang/blob/master/standard/binary.md)
* `shasum --algorithm 256 --binary` hashes the bytes of the CBOR representation to give us the final digest

With this digest in hand we can safely begin to refactor.

## Simplification

The previous chapter concluded with the following Dhall configuration file containing several repetitive backport-related rules:

```dhall
{ pull_request_rules =
    [ { actions =
          { backport = None { branches : Optional (List Text) }
          , delete_head_branch = None {}
          , label =
              None { add : Optional (List Text), remove : Optional (List Text) }
          , merge =
              Some
                { method = Some < merge | rebase | squash >.squash
                , rebase_fallback = None < merge | null | squash >
                , strict = Some < dumb : Bool | smart >.smart
                , strict_method = None < merge | rebase >
                }
          }
      , conditions =
          [ "status-success=continuous-integration/appveyor/pr"
          , "label=merge me"
          , "#approved-reviews-by>=1"
          ]
      , name = "Automatically merge pull requests"
      }
    , { actions =
          { backport = None { branches : Optional (List Text) }
          , delete_head_branch = Some {=}
          , label =
              None { add : Optional (List Text), remove : Optional (List Text) }
          , merge =
              None
                { method : Optional < merge | rebase | squash >
                , rebase_fallback : Optional < merge | null | squash >
                , strict : Optional < dumb : Bool | smart >
                , strict_method : Optional < merge | rebase >
                }
          }
      , conditions = [ "merged" ]
      , name = "Delete head branch after merge"
      }
    , { actions =
          { backport = Some { branches = Some [ "1.0.x" ] }
          , delete_head_branch = None {}
          , label =
              Some { add = None (List Text), remove = Some [ "backport-1.0" ] }
          , merge =
              None
                { method : Optional < merge | rebase | squash >
                , rebase_fallback : Optional < merge | null | squash >
                , strict : Optional < dumb : Bool | smart >
                , strict_method : Optional < merge | rebase >
                }
          }
      , conditions = [ "merged", "label=backport-1.0" ]
      , name = "backport patches to 1.0.x branch"
      }
    , ...
    , { actions =
          { backport = Some { branches = Some [ "1.5.x" ] }
          , delete_head_branch = None {}
          , label =
              Some { add = None (List Text), remove = Some [ "backport-1.5" ] }
          , merge =
              None
                { method : Optional < merge | rebase | squash >
                , rebase_fallback : Optional < merge | null | squash >
                , strict : Optional < dumb : Bool | smart >
                , strict_method : Optional < merge | rebase >
                }
          }
      , conditions = [ "merged", "label=backport" ]
      , name = "backport patches to 1.5.x branch"
      }
    ]
}
```

We can reduce this repetition by creating a `backport` function that we can then invoke repeatedly.

We begin by lifting up an example backport rule to the beginning of our configuration file:

```haskell
let backport =
      { actions =
          { backport = Some { branches = Some [ "1.0.x" ] }
          , delete_head_branch = None {}
          , label =
              Some
                { add = None (List Text)
                , remove = Some [ "status:backport-1.0" ]
                }
          , merge =
              None
                { method : Optional < merge | rebase | squash >
                , rebase_fallback : Optional < merge | null | squash >
                , strict : Optional < dumb : Bool | smart >
                , strict_method : Optional < merge | rebase >
                }
          }
      , conditions = [ "merged", "label=status:backport-1.0" ]
      , name = "backport patches to 1.0.x branch"
      }

in  ...
```

The only thing that changes between each backport rule is the version number, so we'll turn `backport` into a function of one argument (the `version` string):

```haskell
let backport =
          \(version : Text)
      ->  { actions =
              { backport = Some { branches = Some [ "${version}.x" ] }
              , delete_head_branch = None {}
              , label =
                  Some
                    { add = None (List Text)
                    , remove = Some [ "status:backport-${version}" ]
                    }
              , merge =
                  None
                    { method : Optional < merge | rebase | squash >
                    , rebase_fallback : Optional < merge | null | squash >
                    , strict : Optional < dumb : Bool | smart >
                    , strict_method : Optional < merge | rebase >
                    }
              }
          , conditions = [ "merged", "label=status:backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

in  { pull_request_rules =
        [ { actions =
              { backport = None { branches : Optional (List Text) }
              , delete_head_branch = None {}
              , label =
                  None
                    { add : Optional (List Text)
                    , remove : Optional (List Text)
                    }
              , merge =
                  Some
                    { method = Some < merge | rebase | squash >.squash
                    , rebase_fallback = None < merge | null | squash >
                    , strict = Some < dumb : Bool | smart >.smart
                    , strict_method = None < merge | rebase >
                    }
              }
          , conditions =
              [ "status-success=continuous-integration/appveyor/pr"
              , "label=merge me"
              , "#approved-reviews-by>=1"
              ]
          , name = "Automatically merge pull requests"
          }
        , { actions =
              { backport = None { branches : Optional (List Text) }
              , delete_head_branch = Some {=}
              , label =
                  None
                    { add : Optional (List Text)
                    , remove : Optional (List Text)
                    }
              , merge =
                  None
                    { method : Optional < merge | rebase | squash >
                    , rebase_fallback : Optional < merge | null | squash >
                    , strict : Optional < dumb : Bool | smart >
                    , strict_method : Optional < merge | rebase >
                    }
              }
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

... which we use above to simplify the last 6 backport-related rules.

Finally, we can reuse the types we defined within our `./schema.dhall` file to simplify the configuration file further:

```haskell
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

let backport =
          \(version : Text)
      ->  { actions =
              { backport = Some { branches = Some [ "${version}.x" ] }
              , delete_head_branch = None DeleteHeadBranch
              , label =
                  Some
                    { add = None (List Text)
                    , remove = Some [ "status:backport-${version}" ]
                    }
              , merge = None Merge
              }
          , conditions = [ "merged", "label=status:backport-${version}" ]
          , name = "backport patches to ${version}.x branch"
          }

in  { pull_request_rules =
        [ { actions =
              { backport = None Backport
              , delete_head_branch = None DeleteHeadBranch
              , label = None Label
              , merge =
                  Some
                    { method = Some Method.squash
                    , rebase_fallback = None RebaseFallback
                    , strict = Some Strict.smart
                    , strict_method = None StrictMethod
                    }
              }
          , conditions =
              [ "status-success=continuous-integration/appveyor/pr"
              , "label=merge me"
              , "#approved-reviews-by>=1"
              ]
          , name = "Automatically merge pull requests"
          }
        , { actions =
              { backport = None Backport
              , delete_head_branch = Some {=}
              , label = None Label
              , merge = None Merge
              }
          , conditions = [ "merged" ]
          , name = "Delete head branch after merge"
          }
        , backport "1.0"
        , backport "1.1"
        , backport "1.2"
        , backport "1.3"
        , backport "1.4"
        ]
    }
```

## Verification

We could convert back to YAML to confirm that the result is the same:

```bash
$ dhall-to-yaml --file ./mergify.dhall
```
```yaml
pull_request_rules:
- actions:
    merge:
      strict: smart
      method: squash
  name: Automatically merge pull requests
  conditions:
  - status-success=continuous-integration/appveyor/pr
  - label=merge me
  - '#approved-reviews-by>=1'
- ...
```

... but we can more easily recompute the semantic integrity check and verify
that the hash remains the same:

```bash
$ dhall hash --file ./mergify.dhall 
sha256:40e5e1ea3553a14ae667e292506f70b74f02501b86cf496753f8f059fd939c2f
```

In fact, we could automate this check every time we generated YAML by including the hash in the command line, like this:

```bash
$ dhall-to-yaml <<< './mergify.dhall sha256:40e5e1ea3553a14ae667e292506f70b74f02501b86cf496753f8f059fd939c2f'
```
```yaml
pull_request_rules:
- actions:
    merge:
      strict: smart
      method: squash
  name: Automatically merge pull requests
  conditions:
  - status-success=continuous-integration/appveyor/pr
  - label=merge me
  - '#approved-reviews-by>=1'
- ...
```

Note that the language currently only supports verifying integrity checks surrounding imported expressions (such as `./mergify.dhall`), but does not yet support verifying integrity checks protecting arbitrary expressions.  The latter feature would allow us to embed the integrity check within `./mergify.dhall`.

If you are interested in the latter feature you can find a draft language proposal to support generalized integrity checks here:

* [Integrity protect any expression](https://github.com/dhall-lang/dhall-lang/pull/668)

## Next steps

Our configuration file is less repetitive than before, but still requires us to specify every possible field (including unused `None`-valued fields).  The next chapter covers how to simplify configuration files like our own with a large number of absent or default-valued fields.
