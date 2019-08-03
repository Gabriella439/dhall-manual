# How to replace an existing YAML configuration file with Dhall

This chapter explains how to simplify a repetitive YAML configuration file by:

* converting the YAML configuration file to an equivalent Dhall configuration file
* using Dhall's programming language features to simplify the configuration
* generating the original YAML from the simpler Dhall configuration file

## The Problem

We'll use a [Mergify](https://mergify.io/) configuration file as our running example, where Mergify is a GitHub App that can automatically merge and backport pull requests.  You configure the app's behavior through a YAML configuration file checked into the base branch of your repository named `.mergify.yml`

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

The above configuration instructs Mergify to automatically:

* "squash merge" pull requests with a `merge me` label and at least one approval
* delete all merged branches
* backport pull requests to release branches if they have a `backport-*` label

This configuration is repetitive because the last five rules are almost identical: each new release branch requires that we copy, paste, and modify the instructions for automatically backporting pull requests of a given label.

We can use the Dhall configuration language to reduce the repetition, but in order to do so we need to first translate the configuration file as literally as possible to Dhall.  After that we can begin using Dhall's language features to remove some of the repetition.

We'll use the `yaml-to-dhall` command-line tool to translate our YAML configuration file to Dhall rather than translating the file by hand (which is error-prone).  However, we need to provide `yaml-to-dhall` with an expected Dhall type in order to perform the translation.  Picking the right Dhall type is not as hard as translating the entire file, but it's also not trivial, so we'll step through the process of selecting an appropriate type.

## Arbitrary YAML

First, you can always decode an arbitrary YAML configuration into a Dhall expression of the following type:

```haskell
https://prelude.dhall-lang.org/JSON/Type
```

... which is a convenient remote import that resolves to:

```haskell
  ∀(JSON : Type)
→ ∀ ( json
    : { array :
          List JSON → JSON
      , bool :
          Bool → JSON
      , null :
          JSON
      , number :
          Double → JSON
      , object :
          List { mapKey : Text, mapValue : JSON } → JSON
      , string :
          Text → JSON
      }
    )
→ JSON
```

The reason that the type says "JSON" is because Dhall uses the same representation for modeling both JSON and YAML, so they share the same type for encoding arbitrary data.

Let's quickly verify that `yaml-to-dhall` can translate our YAML configuration to a Dhall expression of the above type:

```bash
$ yaml-to-dhall --ascii https://prelude.dhall-lang.org/JSON/Type < ./mergify.yml
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
...
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
...
```

This representation looks like a labeled syntax tree for our original YAML file.  For example, the outer node in the syntax tree is an object, where the first key is named `pull_request_rules` and the corresponding value is an array of nested objects.

We can quickly verify that this generated Dhall expression corresponds to a valid YAML file by using `dhall-to-yaml` to perform the reverse translation:

```bash
$ yaml-to-dhall --ascii https://prelude.dhall-lang.org/JSON/Type < ./mergify.yml | dhall-to-yaml
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
...
```

This matches our original YAML configuration, except that the keys are now sorted.  `dhall-to-yaml` [does not yet support](https://github.com/dhall-lang/dhall-haskell/issues/1187) preserving the original field order.

The type `https://prelude.dhall-lang.org/JSON/Type` is a "weak" type, meaning that the type guarantees little about expressions of that type.  For example, our Mergify configuration should always have an outer field named `pull_request_rules`, but you would not be able to infer that fact from the weak type.

## Strong types

We could technically stop there and begin factoring out repetition in the generated Dhall code.  However, I generally recommend translating YAML to a Dhall expression with a "stronger" type when possible.

A "strong" type is the opposite of a "weak" type.  The "stronger" the type, the more guarantees the type provides about values of that type.  Ideally, we would like to "strengthen" the type of our Dhall configuration file until we're left with as little arbitrary YAML as possible.  In the case of a Mergify configuration [the documentation](https://doc.mergify.io/configuration.html) spells out the entire schema, so by the time we're done our expected Dhall type should be free of arbitrary YAML.

For example, the Mergify documentation specifies that a Mergify configuration file always contains a top-level object with only one key named `pull_request_rules` that in turn contained an array of rules.  Armed with that knowledge, we can begin to carve out a stronger type from our initial weak type, like this:

```haskell
-- ./schema.dhall

let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

-- TODO: We'll flesh out the schema for Rule as we learn more
let Rule = JSON/Type

in  { pull_request_rules : List Rule }
```

Here we've created a file named `schema.dhall` to store the expected Dhall type (a.k.a. "the schema") since we don't to cram that entire type on the command line.  We'll keep reusing this file as our schema grows.

`yaml-to-dhall` can then use the above type to produce a more structured Dhall expression:

```bash
$ yaml-to-dhall --ascii ./schema.dhall < ./mergify.yml
```
```haskell
{ pull_request_rules =
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
      ->  ...
...
```

Now the generated expression slightly resembles a more idiomatic Dhall configuration.  The outer record is an ordinary Dhall record and the `pull_request_rules` field contains an ordinary Dhall `List`.  The `Rule`s stored inside that list could still be arbitrary YAML, though, because we haven't narrowed down the type of a `Rule`, yet.
