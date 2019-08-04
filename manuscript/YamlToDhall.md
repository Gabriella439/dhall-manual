# How to convert an existing YAML configuration file to Dhall

This chapter explains how to transform a YAML configuration file into an equivalent Dhall configuration file with the assistance of the `yaml-to-dhall` command-line tool.

## The Problem

We'll use a [Mergify](https://mergify.io/) configuration file as our running example, where Mergify is a GitHub App that can automatically merge and backport pull requests.  You configure the app's behavior through a YAML configuration file named `.mergify.yml` checked into the base branch of your repository.

Let's illustrate the problem with the following representative `.mergify.yml` configuration, which you can skim for now:

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
        method: squash

  - name: Delete head branch after merge
    conditions:
      - merged
    actions:
      delete_head_branch: {}

  - name: backport patches to 1.0.x branch
    conditions:
      - merged
      - label=backport-1.0
    actions:
      backport:
        branches:
          - 1.0.x
      label:
        remove:
          - "backport-1.0"

  - name: backport patches to 1.1.x branch
    conditions:
      - merged
      - label=backport-1.1
    actions:
      backport:
        branches:
          - 1.1.x
      label:
        remove:
          - "backport-1.1"

  - name: backport patches to 1.2.x branch
    conditions:
      - merged
      - label=backport-1.2
    actions:
      backport:
        branches:
          - 1.2.x
      label:
        remove:
          - "backport-1.2"

  - name: backport patches to 1.3.x branch
    conditions:
      - merged
      - label=backport-1.3
    actions:
      backport:
        branches:
          - 1.3.x
      label:
        remove:
          - "backport-1.3"

  - name: backport patches to 1.4.x branch
    conditions:
      - merged
      - label=backport-1.4
    actions:
      backport:
        branches:
          - 1.4.x
      label:
        remove:
          - "backport-1.4"

  - name: backport patches to 1.5.x branch
    conditions:
      - merged
      - label=backport
    actions:
      backport:
        branches:
          - 1.5.x
      label:
        remove:
          - "backport-1.5"
```

The above list of rules instructs Mergify to automatically:

* "squash merge" pull requests with a `merge me` label and at least one approval
* delete all merged branches
* backport pull requests to release branches if they have a `backport-*` label

The last five backport-related rules are nearly identical: each new release branch requires that we copy, paste, and modify the instructions for automatically backporting pull requests of a given label.

We can reduce that repetition, but first we need to translate the configuration file as literally as possible to Dhall.  We can then use Dhall's programming language features to factor out these repetitive rules in a following chapter.

We'll use the `yaml-to-dhall` command-line tool to translate our YAML configuration to a Dhall configuration rather than translating the file by hand (which is error-prone).  However, we need to provide `yaml-to-dhall` with an expected Dhall type in order to perform the translation.  Picking the right Dhall type is not as difficult as translating the entire file, but it's also not trivial, so we'll step through the process of selecting an appropriate type.

## Arbitrary YAML

First, you can always opt to decode an arbitrary YAML configuration into a Dhall expression of the following type:

```haskell
https://prelude.dhall-lang.org/JSON/Type
```

... which is a convenient remote import that evaluates to:

```haskell
    forall (JSON : Type)
->  forall  ( json
            : { array :
                  List JSON -> JSON
              , bool :
                  Bool -> JSON
              , null :
                  JSON
              , number :
                  Double -> JSON
              , object :
                  List { mapKey : Text, mapValue : JSON } -> JSON
              , string :
                  Text -> JSON
              }
            )
->  JSON
```

The reason that the type says "JSON" is because Dhall uses the same representation for modeling both arbitrary JSON and arbitrary YAML, so they share the above type.

Let's quickly verify that `yaml-to-dhall` can translate our Mergify configuration into a sensible Dhall expression:

```bash
$ yaml-to-dhall --ascii https://prelude.dhall-lang.org/JSON/Type < ./mergify.yml \
>     tee ./mergify.dhall
```
```haskell
    \(JSON : Type)
->  \ ( json
      : { array :
            List JSON -> JSON
        , bool :
            Bool -> JSON
        , null :
            JSON
        , number :
            Double -> JSON
        , object :
            List { mapKey : Text, mapValue : JSON } -> JSON
        , string :
            Text -> JSON
        }
      )
->  json.object
      [ { mapKey =
            "pull_request_rules"
        , mapValue =
            json.array
              [ json.object
                  [ { mapKey =
                        "actions"
                    , mapValue =
                        json.object
                          [ { mapKey =
                                "merge"
                            , mapValue =
                                json.object
                                  [ { mapKey =
                                        "strict"
                                    , mapValue =
                                        json.string "smart"
                                    }
                                  , { mapKey =
                                        "method"
                                    , mapValue =
                                        json.string "squash"
                                    }
                                  ]
                            }
                          ]
                    }
                  , ...
                  ]
              , ...
              ]
        }
      ]
```

You can better understand the generated Dhall expression by ignoring the first 16 lines and reading the remainder of the output:

```haskell
...
->  json.object
      [ { mapKey =
            "pull_request_rules"
        , mapValue =
            json.array
              [ json.object
                  [ { mapKey =
                        "actions"
                    , mapValue =
                        json.object
                          [ { mapKey =
                                "merge"
                            , mapValue =
                                json.object
                                  [ { mapKey =
                                        "strict"
                                    , mapValue =
                                        json.string "smart"
                                    }
                                  , { mapKey =
                                        "method"
                                    , mapValue =
                                        json.string "squash"
                                    }
                                  ]
                            }
                          ]
                    }
                  , ...
                  ]
              , ...
              ]
        }
      ]
```

This representation looks like a labeled syntax tree for our original YAML file.  For example, the outer node in the syntax tree is an object (i.e. `json.object`), where the first key (i.e. `mapKey`) is named `pull_request_rules` and the corresponding value (i.e. `mapValue`) is an array (i.e. `json.array`) of nested objects.

We can verify that this generated Dhall expression models a valid YAML file by piping the output through the `dhall-to-yaml` command to perform the reverse translation:

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

This matches our original YAML configuration, except that the keys are now sorted.  `dhall-to-yaml` [does not yet support](https://github.com/dhall-lang/dhall-haskell/issues/1187) preserving the original field order.  Fortunately, the meaning of the configuration does not change when reordering fields.

The type `https://prelude.dhall-lang.org/JSON/Type` is a "weak" type, meaning that the type guarantees little about expressions having that type.  For example, our Mergify configuration should always have an outer field named `pull_request_rules`, but you would not be able to infer the presence of that field from the weak type.

## Strong types

We could technically accept that translation to Dhall and begin factoring out repetition in the generated Dhall code.  However, I generally recommend translating YAML to a Dhall expression with a "stronger" type when possible.

A "strong" type is the opposite of a "weak" type.  The "stronger" the type, the more guarantees the type provides about values having that type.  Ideally, we would like to "strengthen" the type of our Dhall configuration file until we're left with as little arbitrary YAML as possible.  In fact, we can remove all arbitrary YAML from our configuration because Mergify documents their configuration schema and we can translate Mergify's documented guarantees into type-level guarantees.

For example, the [Mergify file format documentation](https://doc.mergify.io/configuration.html) specifies that:

> The file main entry is a dictionary whose key is named `pull_request_rules`.

... which we can translate to:

```haskell
-- ./schema.dhall

let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Configuration = { pull_request_rules : JSON/Type }

in  Configuration
```

Here we've created a file named `schema.dhall` to store the expected Dhall type (a.k.a. "the schema") so that we don't to cram that entire type on the command line.  For example, you can test drive the above schema works by
running:

```bash
$ yaml-to-dhall --ascii ./schema.dhall < ./mergify.yml
```
```haskell
{ pull_request_rules =
    ...
}
```

We'll keep adding to this file as we translate the Mergify schema to Dhall.

The Mergify schema then says that:

> ... The value of the `pull_request_rules` key must be a list of dictionary.
>
> Each dictionary must have the following keys:
>
> | Key Name     | Value Type | Value Description |
> |--------------|------------|-------------------|
> | `name`       | ...        | ...               |
> | `conditions` | ...        | ...               |
> | `actions`    | ...        | ...               |

... which translates to this Dhall type:

```haskell
let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Rule = { name : JSON/Type, conditions : JSON/Type, actions : JSON/Type }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

... and if we include the expected value types, we get:

> | Key Name     | Value Type            | Value Description |
> |--------------|-----------------------|-------------------|
> | `name`       | string                | ...               |
> | `conditions` | array of Conditions   | ...               |
> | `actions`    | dictionary of Actions | ...               |

... which corresponds to:

```haskell
let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Condition = JSON/Type

let Action = JSON/Type

let Rule =
      { name :
          Text
      , conditions :
          List Condition
      , actions :
          List { mapKey : Text, mapValue : Action }
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

`yaml-to-dhall` can then use the above type to produce a more structured Dhall expression:

```bash
$ yaml-to-dhall --ascii ./schema.dhall < ./mergify.yml
```
```haskell
{ pull_request_rules =
    [ { actions =
          [ { mapKey =
                "merge"
            , mapValue =
                    \(JSON : Type)
                ->  \ ( json
                      : { array :
                            List JSON -> JSON
                        , bool :
                            Bool -> JSON
                        , null :
                            JSON
                        , number :
                            Double -> JSON
                        , object :
                            List { mapKey : Text, mapValue : JSON } -> JSON
                        , string :
                            Text -> JSON
                        }
                      )
                ->  json.object
                      [ { mapKey = "strict", mapValue = json.string "smart" }
                      , { mapKey = "method", mapValue = json.string "squash" }
                      ]
            }
          ]
      , conditions =
          [     \(JSON : Type)
            ->  \ ( json
                  : { array :
                        List JSON -> JSON
                    , bool :
                        Bool -> JSON
                    , null :
                        JSON
                    , number :
                        Double -> JSON
                    , object :
                        List { mapKey : Text, mapValue : JSON } -> JSON
                    , string :
                        Text -> JSON
                    }
                  )
            ->  json.string "status-success=continuous-integration/appveyor/pr"
          ,     \(JSON : Type)
            ->  \ ( json
                  : { array :
                        List JSON -> JSON
                    , bool :
                        Bool -> JSON
                    , null :
                        JSON
                    , number :
                        Double -> JSON
                    , object :
                        List { mapKey : Text, mapValue : JSON } -> JSON
                    , string :
                        Text -> JSON
                    }
                  )
            ->  json.string "label=merge me"
          ,     \(JSON : Type)
            ->  \ ( json
                  : { array :
                        List JSON -> JSON
                    , bool :
                        Bool -> JSON
                    , null :
                        JSON
                    , number :
                        Double -> JSON
                    , object :
                        List { mapKey : Text, mapValue : JSON } -> JSON
                    , string :
                        Text -> JSON
                    }
                  )
            ->  json.string "#approved-reviews-by>=1"
          ]
      , name =
          "Automatically merge pull requests"
      }
...
```

Now the generated expression begins to resemble an idiomatic Dhall configuration.  The outer record is an ordinary Dhall record and the `pull_request_rules` field of that record contains an ordinary Dhall `List`.  However, the "leaves" of our expression are still cluttered by arbitrary YAML that we need to narrow down with more specific types.

## Keep it simple

What should the type of a `Condition` be?

Technically, a `Condition` is just a string as far as the YAML configuration is concerned.  However, Mergify does not accept arbitrary strings for conditions and the documentation has an entire section devoted to the [grammar for `Condition` strings](https://doc.mergify.io/conditions.html#conditions).

Reagrdless, we're going to ignore that grammar for now and treat a `Condition` as `Text`:

```haskell
let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Condition = Text

let Action = JSON/Type

let Rule =
      { name :
          Text
      , conditions :
          List Condition
      , actions :
          List { mapKey : Text, mapValue : Action }
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

Why?  Because our first priority is getting rid of arbitrary YAML.  We can strengthen the type of `Condition` later on.

In general, try to [keep it simple](https://en.wikipedia.org/wiki/KISS_principle) when first adapting a YAML configuration to Dhall, otherwise you'll lose steam quickly.  Remember: our original goal was to simplify our Mergify configuration.

## Essential complexity

Now we only need to eliminate one remaining source of arbitrary YAML: `Action`s.  So far we used a list of key-value pairs to represent `actions` because the documentation said that `actions` is a dictionary.  However, if dig deeper the [documentation for `Action`s](https://doc.mergify.io/actions.html) says that only the following keys are allowed:

* `assign`
* `backport`
* `comment`
* `close`
* `delete_head_branch`
* `dismiss_reviews`
* `label`
* `merge`
* `request_reviews`

... although we'll only consider keys that we actually use to keep this simple.

Moreover, each of the above keys is associated with a different type of value:  

> ### `backport`
>
> The `backport` action takes the following parameter:
>
> | Key name   | Value type      | Default | Value Description |
> |------------|-----------------|---------|-------------------|
> | `branches` | array of string | ...     | ...               |
>
> ### `delete_head_branch`
>
> This action takes no configuration options
>
> ### `label`
>
> | Key name | Value type      | Default | Value Description |
> |----------|-----------------|---------|-------------------|
> | `add`    | array of string | ...     | ...               |
> | `remove` | array of string | ...     | ...               |
> 
> ### `merge`
>
> The `merge` action takes the following parameter:
>
> | Key name          | Value type       | Default | Value Description |
> |-------------------|------------------|---------|-------------------|
> | `method`          | string           | ...     | ...               |
> | `rebase_fallback` | string           | ...     | ...               |
> | `strict`          | boolean or smart | ...     | ...               |
> | `strict_method`   | string           | ...     | ...               |

... and each key has a default value, meaning that the key might not even be present.

This is an example of "essential complexity": the Mergify schema just happens to be complicated and here there is no simple type we can use as an alternative to arbitrary YAML.  Even so, we can still encode a Dhall type for the parts of the schema that we care about.

We begin by encoding the schema for each action.  Each field with a default value (i.e. all of them) must be made `Optional` so that the user can specify that they prefer the default:

```haskell
let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Condition = Text

let Backport = { branches : Optional (List Text) }

let DeleteHeadBranch = {}

let Label = { add : Optional (List Text), remove : Optional (List Text) }

let Merge =
      { method :
          Optional Text
      , rebase_fallback :
          Optional Text
      , strict :
          Optional < Dumb : Bool | Smart >
      , strict_method :
          Optional Text
      }

let Action = JSON/Type

let Rule =
      { name :
          Text
      , conditions :
          List Condition
      , actions :
          List { mapKey : Text, mapValue : Action }
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

Then we can stick those individual action types inside of a record instead of representing them as a list of key-value pairs:

```haskell
let Condition = Text

let Backport = { branches : Optional (List Text) }

let DeleteHeadBranch = {}

let Label = { add : Optional (List Text), remove : Optional (List Text) }

let Merge =
      { method :
          Optional Text
      , rebase_fallback :
          Optional Text
      , strict :
          Optional < dumb : Bool | smart >
      , strict_method :
          Optional Text
      }

let Rule =
      { name :
          Text
      , conditions :
          List Condition
      , actions :
          { backport :
              Optional Backport
          , delete_head_branch :
              Optional DeleteHeadBranch
          , label :
              Optional Label
          , merge :
              Optional Merge
          }
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

We wrap each type of action in an `Optional` field to indicate that they might not be present.

This completes our schema, which is the final type we will use for converting our YAML to Dhall:

```bash
$ yaml-to-dhall --ascii ./schema.dhall < ./mergify.yml \
>     | tee mergify.dhall
```
```haskell
{ pull_request_rules =
    [ { actions =
          { backport =
              None { branches : Optional (List Text) }
          , delete_head_branch =
              None {}
          , label =
              None { add : Optional (List Text), remove : Optional (List Text) }
          , merge =
              Some
              { method =
                  Some "squash"
              , rebase_fallback =
                  None Text
              , strict =
                  Some < dumb : Bool | smart >.smart
              , strict_method =
                  None Text
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
              None { branches : Optional (List Text) }
          , delete_head_branch =
              Some {=}
          , label =
              None { add : Optional (List Text), remove : Optional (List Text) }
          , merge =
              None
                { method :
                    Optional Text
                , rebase_fallback :
                    Optional Text
                , strict :
                    Optional < dumb : Bool | smart >
                , strict_method :
                    Optional Text
                }
          }
      , conditions =
          [ "merged" ]
      , name =
          "Delete head branch after merge"
      }
    , { actions =
          { backport =
              Some { branches = Some [ "1.0.x" ] }
          , delete_head_branch =
              None {}
          , label =
              Some
              { add =
                  None (List Text)
              , remove =
                  Some [ "status:backport-1.0" ]
              }
          , merge =
              None
                { method :
                    Optional Text
                , rebase_fallback :
                    Optional Text
                , strict :
                    Optional < dumb : Bool | smart >
                , strict_method :
                    Optional Text
                }
          }
      , conditions =
          [ "merged", "label=status:backport-1.0" ]
      , name =
          "backport patches to 1.0.x branch"
      }
    , ...
    ]
}
```

Our generated Dhall configuration is pretty big, but in the next chapter we'll begin to simplify this configuration file to remove the repetition.  Once we're done the Dhall configuration will be smaller than the original YAML.

## `dhall-to-yaml`

If we convert our Dhall configuration back to YAML we will get several gratuitous `null`s:

```bash
$ dhall-to-yaml --file ./mergify.dhall
```
```yaml
pull_request_rules:
- actions:
    backport: null
    delete_head_branch: null
    merge:
      strict: smart
      rebase_fallback: null
      method: squash
      strict_method: null
    label: null
  name: Automatically merge pull requests
  conditions:
  - status-success=continuous-integration/appveyor/pr
  - label=merge me
  - '#approved-reviews-by>=1'
- ...
```

We can remove these `null`s using the `--omitNull` flag:

```bash
$ dhall-to-yaml --omitNull --file ./mergify.dhall
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
