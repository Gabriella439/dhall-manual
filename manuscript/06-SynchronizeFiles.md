# How to keep a generated file in sync with the original Dhall expression

Up until now we've been generating a YAML configuration file from a Dhall
configuration file and (presumably) storing both of them in version control.
After all, Mergify requires that the generated YAML configuration file is
stored in version control since Mergify does not directly accept a Dhall
configuration file.  Also, we most likely want to store the Dhall configuration
file in version control, too.

The problem is that nothing guarantees that the Dhall file and the YAML file
remain in sync with one another.  We would like to enforce the invariant that
for every commit to the trunk branch of our repository the YAML file matches
what would be generated from the Dhall configuration file.

This chapter documents how to enforce that invariant using continuous
integration and to set up an ergonomic user experience for keeping the
generated file up-to-date.

> **Note:** I generally advise against storing generated files in version
> control and recommend one of the following alternatives when possible:
>
> * Updating a tool to natively accept Dhall configuration files in addition to
>   JSON / YAML
>
> * Generating a derived JSON / YAML file "on the fly" at build time, deploy
>   time, or runtime
>
> However, these are not always an option, especially when generating
> configuration files used by continuous integration services (e.g. Travis or
> CircleCI)

## The generation script

First, you need to create a script to generate the derived file(s) and that
script to version control.  For example, in our case the script would be:

```bash
#!/bin/sh

# ./generate.sh

dhall-to-yaml --file ./mergify.dhall > ./.mergify.yml
```

This generation script needs to remain under version control because the logic
for generating files might change over time.  You want to ensure that the
generation logic remains in sync with the current branch that a developer is
working on or that CI is testing.

If the script generates a directory or directory tree then you may want to
instead generate the files in two steps:

* Write the files to an intermediate temporary directory
* Use `rsync --delete --archive` to copy from the temporary directory to
  the destination directory

This ensures that files are properly deleted from the destination directory if
they are no longer generated.

## Continuous integration

You will then need to set up CI to perform the following steps:

* Run the generation script in a clean checkout of your source repository
* Fail if any files have changed (e.g. using `git diff --exit-code` for Git)

That minimally ensures that the trunk branch of your repository maintains the
invariant that the generated file remains synchronized with the upstream
Dhall configuration file.

## `git` - pre-commit hooks

You want to be careful about relying exclusively on CI to catch mistakes like
these.  Catching mistakes in CI is still better than catching them in
production, but there are still issues with relying exclusively on CI to catch
mistakes:

* The feedback loop on CI might be too slow for catching mistakes like these

* Small "fixup" commits to fix mistakes caught this way can waste CI resources

* Developers will prefer not having to remember to generate these files at all

The most popular version control tool (`git`) supports "pre-commit" hooks that
you can use to automate the process of keeping generated files in sync.

For example, if you save the following file to the `.git/hooks/pre-commit` path
in your local checkout:

```bash
#!/bin/sh
  
./generate.sh         # Replace this with the path to your generation script

git add .mergify.yml  # Replace this with the path to your generated file(s)

exit
```

... and make the file executable, then `git` will take care of keeping the
generated file(s) up-to-date when ever you create a commit.

Note that the generation script should ideally be quick since this will run
every time your developers create a commit.  This implies that you need to take
care to create a Dhall configuration file that is cheap to interpret if you
go this route.

Fortunately, our Mergify Dhall configuration only takes milliseconds to
interpret, so a pre-commit hook would be appropriate for this example.

The `.git/hooks` directory is unfortunately not kept under version control, but
you can follow these instructions to use hook files that are under version
control:

* [Two Ways to Share Git Hooks with Your Team](https://www.viget.com/articles/two-ways-to-share-git-hooks-with-your-team/)
