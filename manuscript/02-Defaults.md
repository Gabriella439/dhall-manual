# How to support default values

This chapter describes the common idiom of defining a record of default values, using the Mergify configuration file from the end of the previous chapter as an illustrative example.

## The Problem

Most configuration files let you omit options in order to specify that you prefer their default values.

For example, suppose that we want to amend following Mergify rule to instead select the default merge method, whatever it is:

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
        method: squash  # We want to change this to the default merge method
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

In particular, we might not want to explicitly set the field to the current default:

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
        method: merge  # The default is "merge" at the time of this writing
```

Configuration files should preserve the user's intent and if we intend to the default then we should omit the field.  Omission ensures that we continue to track the default value if the default ever changes.

Unfortunately, we cannot add or remove fields from Dhall records without changing their type, so the equivalent Dhall configuration needs some way to accommodate that change within a fixed type.

## `Optional` fields

You can model default-valued fields as `Optional` fields of a record.  For example, in the previous chapter we settled on this strongly-typed representation for merge-related options:

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

... which means that a YAML record like this:

```yaml
        strict: smart
        method: squash
```

... corresponds to this Dhall record:

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
