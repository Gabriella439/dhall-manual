# How to replace an existing YAML configuration file with Dhall

## The Problem

This chapter explains how to simplify a repetitive YAML configuration file by
generating YAML from a Dhall configuration file.

We'll use a [Mergify](https://mergify.io/) configuration file as our running
example.  Mergify is a GitHub App that can automatically merge and backport
pull requests and you configure the app's behavior through a `.mergify.yml`
configuration file checked into the base branch of your repository.

Here's the Mergify configuration that we'll begin with:

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

This configuration file is repetitive because each new release branch we add
requires that we copy, paste, and tweak a matching set of instructions for
automatically backporting pull requests of a given label.

We can't directly modify the merge bot to require less repetition since Mergify
provides the bot using a software-as-a-service model.  Even if we were paying
customers we probably wouldn't know what to request: most programmers are not
qualified to design a domain-specific language embedded inside of YAML.

We can use the Dhall configuration language to reduce the repetition, but in
order to do so we need to first translate the configuration file as literally
as possible to Dhall.  After that we can begin using Dhall's language features
to remove some of the repetition.
