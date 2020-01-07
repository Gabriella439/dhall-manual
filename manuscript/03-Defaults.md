# How to simplify records with many default-valued fields

This chapter describes the language's support for records with default-valued fields, using the Mergify configuration file from the end of the previous chapter as an illustrative example.

## The Problem

Most configuration files let you omit options if you want to specify that you prefer their default values.

For example, suppose we want to change our configuration file to select the default merge method:

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
        # No more `method` field
```

Configuration files should preserve the user's intent and if we intend to select the default then we should omit the field.  Omission ensures that we track possible future changes to the default value.

We can achieve a similar result by using the record completion operator, denoted by two colons: `::`.  This operator extends a record with default values for any missing fields so that we only need to specify non-default values.  You can preview how this operator works using the following excerpt from this chapter's result:

```haskell
...

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

The record completion operator does not guess what the default values should be.  We must specify the defaults for each record type before we can begin to use this operator.

## Record completion

The record completion operator takes two arguments:

```haskell
Schema::fields
```

... and the operator is "syntactic sugar", meaning that the above expression
"desugars" or "expands out" to the following expression:

```haskell
(Schema.default // fields) : Schema.Type
```

... where `//` is the record override operator.

For example, this expression:

```haskell
let Point =
        { Type = { x : Double, y : Double }, default = { x = 0.0, y = 0.0 } }

in  Point::{ x = 1.0 }
```

... is the same thing as:

```haskell
let Point =
        { Type = { x : Double, y : Double }, default = { x = 0.0, y = 0.0 } }

in  (Point.default // { x = 1.0 }) : Point.Type
```

... which in turn is the same thing as:

```haskell
({ x = 0.0, y = 0.0 } // { x = 1.0 }) : { x : Double, y : Double }
```

... which evaluates to:

```haskell
{ x = 1.0, y = 0.0 }
```

In the expression `Schema::fields`, the left argument to the operator (e.g. `Schema`) is a record of following shape:

```haskell
{ Type    = ... -- The type of the record to auto-complete
, default = ... -- Default values for each field
}
```

... and the right argument to the operator (e.g. `fields`) is a record that overrides the `Schema.default` record and is then checked against the `Schema.Type`.

Carefully note that `Schema.default` need not specify default values for each field of `Schema.Type`.  For example, this is a valid schema:

```haskell
let Person =
        { Type = { name : Text, alive : Bool }, default = { alive = True } }

in  Person::{ name = "John Doe" }
```

For the `Person` schema the `name` field is a required field and `alive` is not a required field because we specify a default value for `alive`.

This is why the schema includes the expected type, so that we get a type error if we omit any required fields.  For example, the following expression:

```haskell
let Person =
        { Type = { name : Text, alive : Bool }, default = { alive = True } }

in  Person::{ alive = False }
```

... fails with this type error:

```
Error: Expression doesn't match annotation

{ - name : …
, …
}

4│     Person::{ alive = False }
```

... because we omitted the required `name` field.

## Default `Optional` fields

We can use the record completion operator to simplify our Mergify example by
specifying a default value of `None T` for every field of type `Optional T`,
like this:

```haskell
...
  
let Backport =
      { Type = { branches : Optional (List Text) }
      , default = { branches = None (List Text) }
      }

let DeleteHeadBranch = { Type = {}, default = {=} }

let Label =
      { Type = { add : Optional (List Text), remove : Optional (List Text) }
      , default = { add = None (List Text), remove = None (List Text) }
      }

...

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

...
```

Note that whenever we change a type into a schema we need to change all references to that type to add a `.Type` to the end.  For example, the type of the `backport` field changes from `Optional Backport` to `Optional Backport.Type`.  If you forget to do so then you will get an error message similar to this one:

```
Error: Wrong type of function argument

- Type
+ { … : … }

40│                        Optional Backport
```

... because `Backport` is now a record containing the original `Type`.

## Default non-`Optional` fields

Not all default-valued fields need to be `Optional`.  For example, our original `Rule` type:

```haskell
...

let Rule = { name : Text, conditions : List Condition, actions : Actions.Type }

...
```

... has no `Optional` fields, but we can still sensibly define default values for the `conditions` and `actions` fields:

* The default value for `conditions` is the empty `List`
* The default value for `actions` is the default `Actions` type

... which looks like this:

```haskell
Rule =
      { Type =
          { name : Text, conditions : List Condition, actions : Actions.Type }
      , default =
          { conditions = [] : List Condition, actions = Actions.default }
      }
```

Here the `name` field is required because we do not specify a default value for that field.

## Example usage

The following configuration illustrates the complete example containing both the schema definitions and their use with the record completion operator:

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

... and now we can omit the `Merge` `method` by deleting this line:

```haskell
...

in  { pull_request_rules =
        [ Rule::{
          , actions =
              Actions::{
              , merge =
                  Some
                    Merge::{
                    -- , method = Some Method.squash
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

... and the generated YAML configuration will then select the default `Merge`
`method`.

## Next steps

The final record at the end of the file is now more concise, but as a result the preceding schema definitions now dwarf the actual program configuration.  Fortunately, these schema definitions are reusable across all Mergify configuration files, so we can factor them out into a separate package that we can share with others.  Or we can tuck them away in a separate file even if we don't plan to share them, so that we can focus on the program configuration.

The next chapter covers how to factor out definitions like these into separate files and organize them to follow an opinionated project layout.
