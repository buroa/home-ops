# Rendering a HelmRelease with flate

A render shows the Kubernetes resources that a chart produces. Run flate
from the repository root. A namespace groups resources in Kubernetes.
Use `-n` to select the namespace of the HelmRelease.

A merge base is a shared commit that Git uses to compare branches. If a
chart version or HelmRelease values change, compare the pull request
render with the merge-base render.

```
flate diff hr '<name>' -n '<namespace>' --base '<merge-base-sha>' --no-progress -o diff
```

If a local clone has Git history, use `git merge-base HEAD '<target-ref>'`.
In kritika, run the first `gh` command to get the base and head SHAs. A SHA
identifies a Git commit. The base comes from the target branch. The head
is the commit under review. Use the two SHAs in the second `gh` command
to get the merge base.

```
gh api 'repos/<owner>/<repo>/pulls/<number>' --jq '{base: .base.sha, head: .head.sha}'
gh api 'repos/<owner>/<repo>/compare/<base-sha>...<head-sha>' --jq '.merge_base_commit.sha'
```

Kritika writes the head files to a checkout, a local copy of the files.
The `origin` remote points to the repository on the server. The checkout
has a `.git` directory with that remote and no commits. If the checkout
lacks the merge base, flate fetches it from `origin`.

If you cannot get the merge-base SHA, pass the target branch to `--base`.
Say that the diff compares against the latest commit on that branch.

The baseline is the version that flate compares with the head. If the
checkout lacks Git metadata, look for the baseline files in a separate
directory. If you have those files, pass their directory to `--path-orig`
in place of `--base`.

If you cannot get a baseline, render the head with this command. State
that the review covers the head render only.

```
flate build hr '<name>' -n '<namespace>' --no-progress
```

If the render stays the same, flate can exit with a nonzero status and
show `no HelmRelease named "<name>"`. If you see that error, make sure
that the name and namespace are correct. Read the other messages before
you conclude that the render stayed the same.

If a change causes the HelmRelease under review to fail to render, report
the failure on the changed line. For example, the chart schema can reject
a value. A template can fail with the values of this repository. If flate
fails to fetch a source, say which render you cannot inspect.

The render estimates what Flux applies to the cluster. flate renders
without access to the cluster. The render includes the patches that Flux
applies from a parent Kustomization.

A CRD defines a Kubernetes resource type. flate leaves out CRDs and
Secrets. If a template uses the Kubernetes version, flate supplies its
bundled version.

Use the rendered labels, resource names, and ports to find what depends
on the change. If the render is too long, use `--only-kinds <Kind>,<Kind>`
to show the resource types you need. If only the head renders, use
upstream sources or references in this repository to find the old output.
