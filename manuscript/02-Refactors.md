# How to safely refactor a configuration file

The Dhall language and tooling are optimized for "maintainability" and one
aspect of that is ease of refactoring existing configuration files.

In this chapter we will use functions to reduce repetition and also verify that our refactor is behavior-preserving along the way, beginning from the following Dhall configuration file we generated at the end of the previous chapter:

```haskell
-- ./mergify.dhall

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
    , { actions =
          { backport = Some { branches = Some [ "1.1.x" ] }
          , delete_head_branch = None {}
          , label =
              Some { add = None (List Text), remove = Some [ "backport-1.1" ] }
          , merge =
              None
                { method : Optional < merge | rebase | squash >
                , rebase_fallback : Optional < merge | null | squash >
                , strict : Optional < dumb : Bool | smart >
                , strict_method : Optional < merge | rebase >
                }
          }
      , conditions = [ "merged", "label=backport-1.1" ]
      , name = "backport patches to 1.1.x branch"
      }
    , { actions =
          { backport = Some { branches = Some [ "1.2.x" ] }
          , delete_head_branch = None {}
          , label =
              Some { add = None (List Text), remove = Some [ "backport-1.2" ] }
          , merge =
              None
                { method : Optional < merge | rebase | squash >
                , rebase_fallback : Optional < merge | null | squash >
                , strict : Optional < dumb : Bool | smart >
                , strict_method : Optional < merge | rebase >
                }
          }
      , conditions = [ "merged", "label=backport-1.2" ]
      , name = "backport patches to 1.2.x branch"
      }
    , { actions =
          { backport = Some { branches = Some [ "1.3.x" ] }
          , delete_head_branch = None {}
          , label =
              Some { add = None (List Text), remove = Some [ "backport-1.3" ] }
          , merge =
              None
                { method : Optional < merge | rebase | squash >
                , rebase_fallback : Optional < merge | null | squash >
                , strict : Optional < dumb : Bool | smart >
                , strict_method : Optional < merge | rebase >
                }
          }
      , conditions = [ "merged", "label=backport-1.3" ]
      , name = "backport patches to 1.3.x branch"
      }
    , { actions =
          { backport = Some { branches = Some [ "1.4.x" ] }
          , delete_head_branch = None {}
          , label =
              Some { add = None (List Text), remove = Some [ "backport-1.4" ] }
          , merge =
              None
                { method : Optional < merge | rebase | squash >
                , rebase_fallback : Optional < merge | null | squash >
                , strict : Optional < dumb : Bool | smart >
                , strict_method : Optional < merge | rebase >
                }
          }
      , conditions = [ "merged", "label=backport-1.4" ]
      , name = "backport patches to 1.4.x branch"
      }
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

## Semantic integrity checks

Before starting our refactor we can create a digest (a.k.a. a checksum) of the result, like this:

```bash
$ dhall hash --file ./mergify.dhall 
sha256:c231ab5f3e5694cd8cf8e467255a8f72f91b6d5ff83d90ebbeba4b7dfbbb7df6
```

If our refactor does not change the result then the digest should remain the same.

This digest is known as a "semantic integrity check" or a "semantic hash" for short.  This is not a textual hash, but is instead a hash of the CBOR encoding of the syntax tree in a canonical normal form.

In other words, you could obtain the same hash by chaining the following steps:

```bash
$ dhall --alpha --file ./mergify.dhall | dhall encode | shasum --algorithm 256 --binary
c231ab5f3e5694cd8cf8e467255a8f72f91b6d5ff83d90ebbeba4b7dfbbb7df6 *-
```

... where:

* `dhall --alpha` reduces an expression to a canonical normal form, free of imports and also free of indirection
* `dhall encode` encodes the expression according to a [standard CBOR representation](https://github.com/dhall-lang/dhall-lang/blob/master/standard/binary.md)
* `shasum --algorithm 256 --binary` hashes the bytes of the CBOR representation to give us the final digest

After recording this digest we can safely begin to refactor.

If you have the `watch` command installed, you can track the digest live as you refactor by running:

```
$ watch 'dhall hash --file ./mergify.dhall'
```

## Simplification

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
                , remove = Some [ "backport-1.0" ]
                }
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
                    , remove = Some [ "backport-${version}" ]
                    }
              , merge =
                  None
                    { method : Optional < merge | rebase | squash >
                    , rebase_fallback : Optional < merge | null | squash >
                    , strict : Optional < dumb : Bool | smart >
                    , strict_method : Optional < merge | rebase >
                    }
              }
          , conditions = [ "merged", "label=backport-${version}" ]
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
                    , remove = Some [ "backport-${version}" ]
                    }
              , merge = None Merge
              }
          , conditions = [ "merged", "label=backport-${version}" ]
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
        , backport "1.5"
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
that the digest remains the same:

```bash
$ dhall hash --file ./mergify.dhall 
sha256:ccfbb3688d22cfd0e6506c51e760427ea2925f251b1759654cc8d130db5b9b96
```

Wait a second...  This is not the same digest as before!

How can we tell where things went wrong?

## Comparing two expressions

If we save our original configuration file to `./old.dhall` and our new
configuration file to `./new.dhall` then we can compare the two using the
`dhall diff` command, like this:

```bash
$ dhall diff './old.dhall' './new.dhall'
```
```haskell
{ pull_request_rules = [ …
                       , { conditions = [ …
                                        , "label=backport-1.5"
                                        ]

                         , …
                         }
                       ]

}
```

This is a "semantic diff" meaning that the diff focuses in on only the relevant difference and also includes a "trail of breadcrumbs" showing you the context of the two subexpressions that differ.

The above diff highlights that the final `Rule` inside the `pull_request_rules` `List` differs.  Specifically, the label that the backport rule expects was wrong in the original configuration; the label should have been `backport-1.5` but somebody copying-and-pasting configuration sections mistakenly edited the label to be `backport`.  If you run the diff command yourself you will notice that the `dhall diff` command highlights the `-1.5` suffix in green to further narrow down the difference.

This illustrates the importance of DRY ("Don't Repeat Yourself"): removing repetition often also removes bugs.

## Next steps

Our configuration file is less repetitive than before, but still requires us to specify every possible field (including unused `None`-valued fields).  The next chapter covers how to simplify configuration files like our own with a large number of absent or default-valued fields.
