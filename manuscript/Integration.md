# How to replace an existing YAML configuration file with Dhall

## The Problem

This chapter explains how to simplify a repetitive YAML configuration file by generating YAML from a Dhall configuration file.

We'll use a [Mergify](https://mergify.io/) configuration file as our running example.  Mergify is a GitHub App that can automatically merge and backport pull requests and you configure the app's behavior through a `.mergify.yml` configuration file checked into the base branch of your repository.

Here's a Mergify configuration that we'll begin with:

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
      - label=status:backport-1.0
    actions:
      backport:
        branches:
          - 1.0.x
      label:
        remove:
          - "status:backport-1.0"

  - name: backport patches to 1.1.x branch
    conditions:
      - merged
      - label=status:backport-1.1
    actions:
      backport:
        branches:
          - 1.1.x
      label:
        remove:
          - "status:backport-1.1"

  - name: backport patches to 1.2.x branch
    conditions:
      - merged
      - label=status:backport-1.2
    actions:
      backport:
        branches:
          - 1.2.x
      label:
        remove:
          - "status:backport-1.2"

  - name: backport patches to 1.3.x branch
    conditions:
      - merged
      - label=status:backport-1.3
    actions:
      backport:
        branches:
          - 1.3.x
      label:
        remove:
          - "status:backport-1.3"

  - name: backport patches to 1.4.x branch
    conditions:
      - merged
      - label=status:backport-1.4
    actions:
      backport:
        branches:
          - 1.4.x
      label:
        remove:
          - "status:backport-1.4"

  - name: backport patches to 1.5.x branch
    conditions:
      - merged
      - label=status:backport
    actions:
      backport:
        branches:
          - 1.5.x
      label:
        remove:
          - "status:backport-1.5"
```

The above configuration instructs Mergify to automatically:

* squash merge pull requests with the `merge me` label and at least one approval
* delete merged branches
* backport certain labeled pull requests to release branches

This configuration file is repetitive because each new release branch we add requires that we copy, paste, and tweak a matching set of instructions for automatically backporting pull requests of a given label.

We can use the Dhall configuration language to reduce the repetition, but in order to do so we need to first translate the configuration file as literally as possible to Dhall.  After that we can begin using Dhall's language features to remove some of the repetition.

We'll use the `yaml-to-dhall` command-line tool to translate our YAML configuration file to Dhall rather than translating the file by hand (which is error-prone).  However, we need to provide `yaml-to-dhall` with an expected Dhall type in order to perform the translation.

Assume that the YAML file is sufficiently messy and large that we don't know in advance what Dhall type use.  That's fine; we can discover the correct type by trial and error, using the above YAML configuration file as an example.

Let's begin by telling `yaml-to-dhall` to decode into an empty record type (i.e. `{}`).  We know this expected type is wrong but we can iterate on the type as we go:

```
$ yaml-to-dhall '{}' < ./mergify.yml

Error: Key(s) pull_request_rules present in the YAML object but not in the expected Dhall record type. This is not allowed unless you enable the --records-loose flag:

Expected Dhall type:
{}

YAML:
pull_request_rules:
- actions:
    merge:
      strict: smart
      method: squash
  name: Automatically merge pull requests
  conditions:
  - status-success=continuous-integration/appveyor/pr
  - label=merge me
  - ! '#approved-reviews-by>=1'
...
```

The error message indicates that we asked for an empty record, but the matching section of the YAML configuration contains an unexpected field named `pull_request_rules`.  We can update the expected type to reflect that:

```
$ yaml-to-dhall '{ pull_request_rules : {} }' < ./mergify.yml

Error: Dhall type expression and json value do not match:

Expected Dhall type:
{}

YAML:
- actions:
    merge:
      strict: smart
      method: squash
  name: Automatically merge pull requests
  conditions:
  - status-success=continuous-integration/appveyor/pr
  - label=merge me
  - ! '#approved-reviews-by>=1'
```
