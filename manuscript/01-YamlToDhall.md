# How to convert an existing YAML configuration file to Dhall

This chapter explains how to transform a non-trivial YAML configuration file into an equivalent Dhall configuration file with the assistance of the `yaml-to-dhall` command-line tool.  This comes in handy when translating large configuration files, which are tedious to translate by hand.

## The Problem

We'll use a [Mergify](https://mergify.io/) configuration file as our running example, where Mergify is a GitHub App that can automatically merge and backport pull requests.

The Mergify app is configured with a YAML configuration file named `.mergify.yml` that might look like the following representative example:

```yaml
# ./.mergify.yml

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
* delete branches after they are merged
* backport pull requests to release branches if they have the correct `backport-*` label

The last six backport-related rules are nearly identical because we need to copy, paste, and modify the backport-related logic every time we create a new release branch.

We can reduce that repetition, but first we need to translate the configuration file as literally as possible to Dhall.  After that we can use Dhall's programming language features to factor out repetitive rules.

We'll use the `yaml-to-dhall` command-line tool to translate our YAML configuration file into a Dhall configuration file rather than translating the file by hand (which is error-prone).  However, we need to provide `yaml-to-dhall` with an expected Dhall type in order to perform the translation.  Picking the right Dhall type is not as difficult as translating the entire file, but it's also not trivial, so we'll step through the process of selecting an appropriate type.

## Arbitrary YAML

First, you can always decode an arbitrary YAML configuration into a Dhall expression of the following type:

```haskell
https://prelude.dhall-lang.org/JSON/Type
```

... which is a convenient remote import that evaluates to:

```haskell
  ∀(JSON : Type)
→ ∀ ( json
    : { array : List JSON → JSON
      , bool : Bool → JSON
      , null : JSON
      , number : Double → JSON
      , object : List { mapKey : Text, mapValue : JSON } → JSON
      , string : Text → JSON
      }
    )
→ JSON
```

The reason that the type says "JSON" is because Dhall uses the same representation for modeling both arbitrary JSON and arbitrary YAML, so they share the above type.

Let's quickly verify that `yaml-to-dhall` can use that type to translate our Mergify configuration into a Dhall expression:

```bash
$ yaml-to-dhall --ascii https://prelude.dhall-lang.org/JSON/Type < ./.mergify.yml | tee ./mergify.dhall
```
```haskell
    \(JSON : Type)
->  \ ( json
      : { array : List JSON -> JSON
        , bool : Bool -> JSON
        , null : JSON
        , number : Double -> JSON
        , object : List { mapKey : Text, mapValue : JSON } -> JSON
        , string : Text -> JSON
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
                          [ { mapKey = "merge"
                            , mapValue =
                                json.object
                                  [ { mapKey = "strict"
                                    , mapValue = json.string "smart"
                                    }
                                  , { mapKey = "method"
                                    , mapValue = json.string "squash"
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
                          [ { mapKey = "merge"
                            , mapValue =
                                json.object
                                  [ { mapKey = "strict"
                                    , mapValue = json.string "smart"
                                    }
                                  , { mapKey = "method"
                                    , mapValue = json.string "squash"
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

This matches our original YAML configuration, except that the keys are now sorted, which is unavoidable as `dhall-to-yaml` [does not yet support](https://github.com/dhall-lang/dhall-haskell/issues/1187) preserving the original field order.  Fortunately, the meaning of the configuration does not change when reordering fields.

The type `https://prelude.dhall-lang.org/JSON/Type` is a "weak" type, meaning that we cannot guarantee much for expressions having that type.  For example, our Mergify configuration should always have an outer field named `pull_request_rules`, but you would not be able to infer the presence of that field from the type.

## Strong types

We could stop there and begin factoring out repetition in the generated Dhall code.  However, I generally recommend translating YAML to a Dhall expression with a "stronger" type when possible.

A "strong" type is the opposite of a "weak" type.  The "stronger" the type, the more guarantees the type provides about values having that type.  Ideally, we would like to "strengthen" the type of our Dhall configuration file until we're left with as little arbitrary YAML as possible.  In fact, we can remove all arbitrary YAML from our configuration because Mergify documents their entire YAML schema.

For example, the [Mergify file format documentation](https://doc.mergify.io/configuration.html) specifies that:

> The file main entry is a dictionary whose key is named `pull_request_rules`.

... which we can translate to this initial Dhall type:

```haskell
-- ./schema.dhall

let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Configuration = { pull_request_rules : JSON/Type }

in  Configuration
```

Now the outermost type (named `Configuration`) is a record with one field named `pull_request_rules`, but the contents of that field could still be arbitrary YAML.

Here we've created a file named `schema.dhall` to store the expected Dhall type (a.k.a. "the schema") so that we don't have to fit that entire type on the command line.  For example, you can test drive the above schema works by running:

```bash
$ yaml-to-dhall --ascii ./schema.dhall < ./.mergify.yml
```
```haskell
{ pull_request_rules =
    ...
}
```

We'll continue to expand upon this `./schema.dhall` file as we store what we learn about the Mergify schema in the equivalent Dhall type.

The Mergify schema also says that:

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

          -- Translate the key names to record field names
          -- ↓                 ↓                       ↓
let Rule = { name : JSON/Type, conditions : JSON/Type, actions : JSON/Type }
          --        ↑                       ↑                    ↑
          -- Model the corresponding record values as `JSON/Type` for now

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
             -- ↑ We still do not know what a `Condition` could be

let Action = JSON/Type
          -- ↑ We also still do not know what an `Action` could be

let Rule =
      { name       : Text
                  -- ↑ "string"
      , conditions : List Condition
                  -- ↑ "array of Conditions"
      , actions    : List { mapKey : Text, mapValue : Action }
                  -- ↑ "dictionary of Actions"
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

`yaml-to-dhall` can then use the above type to produce a more structured Dhall expression:

```bash
$ yaml-to-dhall --ascii ./schema.dhall < ./.mergify.yml
```
```haskell
{ pull_request_rules =
    [ { actions =
          [ { mapKey = "merge"
            , mapValue =
                    \(JSON : Type)
                ->  \ ( json
                      : { array : List JSON -> JSON
                        , bool : Bool -> JSON
                        , null : JSON
                        , number : Double -> JSON
                        , object :
                            List { mapKey : Text, mapValue : JSON } -> JSON
                        , string : Text -> JSON
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
                  : { array : List JSON -> JSON
                    , bool : Bool -> JSON
                    , null : JSON
                    , number : Double -> JSON
                    , object : List { mapKey : Text, mapValue : JSON } -> JSON
                    , string : Text -> JSON
                    }
                  )
            ->  json.string "status-success=continuous-integration/appveyor/pr"
          ,     \(JSON : Type)
            ->  \ ( json
                  : { array : List JSON -> JSON
                    , bool : Bool -> JSON
                    , null : JSON
                    , number : Double -> JSON
                    , object : List { mapKey : Text, mapValue : JSON } -> JSON
                    , string : Text -> JSON
                    }
                  )
            ->  json.string "label=merge me"
          ,     \(JSON : Type)
            ->  \ ( json
                  : { array : List JSON -> JSON
                    , bool : Bool -> JSON
                    , null : JSON
                    , number : Double -> JSON
                    , object : List { mapKey : Text, mapValue : JSON } -> JSON
                    , string : Text -> JSON
                    }
                  )
            ->  json.string "#approved-reviews-by>=1"
          ]
      , name = "Automatically merge pull requests"
      }
    , ...
    ]
}
```

Now the generated expression begins to resemble a Dhall configuration we might write by hand.  The outer record is an ordinary Dhall record and the `pull_request_rules` field of that record contains an ordinary Dhall `List`.  However, the "leaves" of our expression are still arbitrary YAML that we need to narrow to more specific types.

So far we've used `JSON/Type` as a type-level `TODO` for our YAML schema that we can insert when we're not sure what some subexpression's type should be.  This trick allows us to incrementally migrate from a weakly-typed configuration to a strongly-typed one.

## Keep it simple

What should the type of a `Condition` be?

```haskell
let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Condition = JSON/Type  -- ← Surely we can do better here

let Action = JSON/Type

let Rule =
      { name : Text
      , conditions : List Condition
      , actions : List { mapKey : Text, mapValue : Action }
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

Technically, a `Condition` is just a string as far as the YAML configuration is concerned.  However, Mergify does not accept arbitrary strings for conditions and the documentation has an entire section devoted to the [grammar for `Condition` strings](https://doc.mergify.io/conditions.html#conditions).

Regardless, we're going to ignore that grammar for now and treat a `Condition` as a synonym for `Text`:

```haskell
let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Condition = Text  -- ← Simple

let Action = JSON/Type

let Rule =
      { name : Text
      , conditions : List Condition
      , actions : List { mapKey : Text, mapValue : Action }
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

Why?  Because our first priority is getting rid of arbitrary YAML.  We can further strengthen the type of `Condition` later on.

In general, try to [keep it simple](https://en.wikipedia.org/wiki/KISS_principle) when first adapting a YAML configuration to Dhall, otherwise you might lose steam and give up on the conversion.  Remember: our goal is to simplify our Mergify configuration and we want to get to work factoring out repetitive code.

## Essential complexity

Now we only need to eliminate one remaining source of arbitrary YAML: `action`s.

```haskell
let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Condition = Text

let Action = JSON/Type

let Rule =
      { name : Text
      , conditions : List Condition
      , actions : List { mapKey : Text, mapValue : Action }
               -- ↑ This field actually has a more precise schema
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

So far we have encoded the `actions` field as a list of key-value pairs because the documentation said that `actions` is a dictionary.  However, if we dig deeper the [documentation for `Action`s](https://doc.mergify.io/actions.html) indicates that this dictionary cannot contain arbitrary key-value pairs.

For example, the dictionary of actions only permits the following keys:

* `assign`
* `backport`
* `comment`
* `close`
* `delete_head_branch`
* `dismiss_reviews`
* `label`
* `merge`
* `request_reviews`

To keep this chapter short, we'll narrow those keys down further to only the keys used within our configuration file:

* `backport`
* `delete_head_branch`
* `label`
* `merge`

Additionally, each of the above keys stores a different type of value:

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
> | Key name          | Value type       | Default | Value Description                                        |
> |-------------------|------------------|---------|----------------------------------------------------------|
> | `method`          | string           | ...     | ...  Possible values are `merge`, `squash` or, `rebase`. |
> | `rebase_fallback` | string           | ...     | ...  Possible values are `merge`, `squash`, `null`.      |
> | `strict`          | boolean or smart | ...     | ...                                                      |
> | `strict_method`   | string           | ...     | ...  Possible values are `merge` or `rebase`.            |


... and every key has a default value, meaning that the user can omit any given key.

This is an example of "essential complexity": the Mergify schema just happens to be complicated, meaning that we cannot use a simple type here.  That's okay, though; we only need to specify a more sophisticated type.

We'll begin with the easy part: encoding the schema for each type of action.  Each field with a default value (i.e. all of them) must be made `Optional` so that the user can omit the value if they want to select the default behavior:

```haskell
let JSON/Type = https://prelude.dhall-lang.org/JSON/Type

let Condition = Text

{-  The `backport` action takes the following parameter:

    | Key name   | Value type      | Default | Value Description |
    |------------|-----------------|---------|-------------------|
    | `branches` | array of string | ...     | ...               |
-}
let Backport = { branches : Optional (List Text) }

--  This action takes no configuration options
let DeleteHeadBranch = {}

{-  | Key name | Value type      | Default | Value Description |
    |----------|-----------------|---------|-------------------|
    | `add`    | array of string | ...     | ...               |
    | `remove` | array of string | ...     | ...               |
-}
let Label = { add : Optional (List Text), remove : Optional (List Text) }

{-  | Key name          | Value type       | Default | Value Description |
    |-------------------|------------------|---------|-------------------|
    | `method`          | string           | ...     | ...               |
    |                   |                  |         | Possible values   |
    |                   |                  |         | are `merge`,      |
    |                   |                  |         | `squash` or,      |
    |                   |                  |         | `rebase`          |
    |                   |                  |         |                   |
    | `rebase_fallback` | string           | ...     | ...               |
    |                   |                  |         | Possible values   |
    |                   |                  |         | are `merge`,      |
    |                   |                  |         | `squash`, `null`. |
    |                   |                  |         |                   |
    | `strict`          | boolean or smart | ...     | ...               |
    |                   |                  |         |                   |
    | `strict_method`   | string           | ...     | ...               |
    |                   |                  |         | Possible values   |
    |                   |                  |         | are `merge` or    |
    |                   |                  |         | `rebase`          |
-}
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

let Action = JSON/Type

let Rule =
      { name : Text
      , conditions : List Condition
      , actions : List { mapKey : Text, mapValue : Action }
      }

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

Then we can stick those individual action types inside of an `Actions` record instead of representing them as a list of key-value pairs:

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

let Configuration = { pull_request_rules : List Rule }

in  Configuration
```

Note that our record of `actions` also has all `Optional` fields, indicating that each type of action may or may not be present for any given `Rule`.  A `Rule` could specify more than one action or not specify any actions at all (which would be useless, but legal).

This completes our schema, which we can now use to convert our YAML to Dhall:

```bash
$ yaml-to-dhall --ascii ./schema.dhall < ./.mergify.yml | tee mergify.dhall
```
```haskell
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
    ]
}
```

The above snippet truncates the output since the generated Dhall configuration starts off large, but now that we have valid Dhall code we can begin using programming features to reduce repetition.

## Simplification

Let's create a `backport` function that we can invoke repeatedly for each backport rule.  We begin by lifting up an example backport rule to the beginning of our configuration file:

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

Finally, we can convert back to YAML to confirm that everything still works:

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
