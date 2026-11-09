---
name: reviewing-pull-requests
description: Use when you review a pull request or a Renovate update in this Flux repository. Also use it to find out what a change under kubernetes/ affects. The skill tells you how to find the upstream project of a dependency and its breaking changes. It also describes the Renovate rules of this repository. It tells you how to search for what depends on a change, and what a check proves. references/registry.md reads images and OCI charts with crane. references/rendering.md renders and diffs HelmReleases with flate.
---

# Reviewing pull requests in home-ops

Flux applies `kubernetes/` from `main` to the cluster. A merge goes live
within minutes. Secrets are ExternalSecrets that come from 1Password.
Judge their key names and mappings from the manifests.

Use only the sections that apply to the pull request:

- A Renovate update: use it for a pull request on a `renovate/` branch.
- A change under `kubernetes/`: use it for any change there. This includes
  a Renovate update that changes files there.
- What a check proves: use it for every review.
- In kritika: if the review runs in kritika, use it.

## A Renovate update

Decide whether the update is safe to merge. Find out what changed upstream
between the old and the new version. Then find out whether anything in this
repository depends on that change. Use these tools to read the sources:

- Read GitHub with `gh`.
- Read the changelog or the documentation site of a project with `curl`.
- If `crane` is available, read an image or an OCI chart from its registry
  with `crane`, as `references/registry.md` describes.

### The update

Get these facts from the title, the body, the labels, and the diff:

- The package and its datasource: container image, Helm chart as an
  `OCIRepository`, GitHub Action, mise tool, Grafana dashboard, or Renovate
  preset.
- The old and the new version or digest.
- The update type, from the `type/*` label.
- The changed files. These are the places that use the dependency.

The body of a Renovate pull request has a collapsed `Release Notes` section
with excerpts from upstream. Read it first. Then go upstream in these cases:

- The section is missing, cut short, or not clear.
- The update is a major update.
- The update skips several versions. Read every release between the old
  and the new version, because breaking changes often come in a middle
  release.

### Finding the upstream

- An image at `<registry>/<owner>/<repo>` usually comes from
  `<owner>/<repo>` on GitHub. If it does not, the pull request body or the
  `org.opencontainers.image.source` label of the image names the source.
  Read the label as `references/registry.md` describes.
- Compare an image update with the image itself. If `crane` is available,
  read the configuration of the pinned digest as `references/registry.md`
  describes. Its version and revision labels show what the image was built
  from. Compare them with the release that you review.
- The source of a chart is the `OCIRepository` of the app
  (`ocirepository.yaml`). Its `url` is the registry path, and its `ref.tag`
  is the version.
- If the chart path is under an owner (`oci://<registry>/<owner>/...`), the
  chart belongs to that owner on GitHub. The `sources` or `home` field in
  `Chart.yaml` names the repository. `crane config` on the chart tag prints
  these fields without a download of the chart.
- If the chart path is under `ghcr.io/home-operations/charts-mirror/<chart>`,
  the chart is a mirror. A mirror is a copy of a chart that its project
  does not publish in OCI form. `apps/<chart>/metadata.yaml` in
  `home-operations/charts-mirror` names the upstream Helm repository. The
  changelog and the releases of the chart are in the GitHub repository of
  that project.
- The chart version and the app version are different. Read the changelog
  of the chart. If `appVersion` changed, also read the changelog of the
  application.
- Some repositories publish many charts. Their release tags start with the
  chart name (`<chart>-1.2.3`, not `v1.2.3`). If you cannot find a release,
  list the releases and look for the chart name.
- A digest-only update does not change the version. Say whether the tag was
  rebuilt, for example for a security fix or because the pin is a
  `latest`-style tag. Also say what it was rebuilt from. To find out,
  compare `created` and `revision` of the old and the new digest.
- If a release note says "see #2785", read the reference with
  `gh issue view` or `gh pr view`. If it is an issue, find the merged pull
  request that closed it
  (`gh pr list -R <repo> --search "#<n>" --state merged`). The pull request
  shows what changed and the scope of the change. The issue and the release
  note sometimes do not.

### What breaks

Flag these items in each release:

- `BREAKING CHANGE`, `⚠`, or `!:` markers.
- Helm values, CRD fields, environment variables, and flags that were
  removed or renamed.
- A new minimum version of Kubernetes, Flux, Talos, or the kernel.
- Schema or data migrations that you cannot reverse.
- Changed defaults, for example authentication, storage class, ports, or
  probes.
- Deprecations that are now errors.
- New required keys that have no default.
- Label values that changed while the key stayed the same, and resource
  names that lost or gained a prefix. These break anything that selects on
  the old value, but a search for the key still finds a match.

Minor and patch updates can also contain breaking changes. The phrase
"Chart name prefix removed" can mean label values or resource names. Find
out which one from the pull request or the template, not from the sentence.

An update can also change images that are not in the diff. For example, a
chart or an operator changes its default images. Or it stops setting them
and lets the built-in defaults of another operator apply. Name each of
these images with its old and its new version. Read the versions from the
source of the chart or the operator at both versions. This repository does
not pin these images, so Renovate never opens a pull request for them. The
review is the only place where they show.

A chart can set a version by default, for example the image of a workload
that its operator manages. That version is the default in the `values.yaml`
of the chart at the new tag, unless the HelmRelease values pin it.

If the update is under `kubernetes/`, render it and find what depends on
it, as [A change under `kubernetes/`](#a-change-under-kubernetes)
describes. A breaking change that affects nothing here is not a finding.
One sentence in the take is enough, with the search that found nothing.

### Renovate's rules

`.renovaterc.json5` defines how Renovate treats this update:

- Its `automerge` rules say whether the pull request merges without a
  person. If it does, the review is the last look before the merge.
- Its `groupName` rules say which pull requests update several packages
  together. The review must cover each of these packages.
- Its labels, and the `!` that the shared preset adds to a major update,
  must agree with the title. If they do not agree, add a note.

## A change under `kubernetes/`

If a chart version or HelmRelease values change, and `flate` is
available, render the HelmRelease as `references/rendering.md` describes.

### Exposure here

Do not ask whether this repository sets the old thing. Ask whether anything
here depends on it. Search `kubernetes/` for these items:

- A label key or value. Search for anything that selects on it: the
  `matchLabels` and `selector` fields of monitors, network policies,
  and disruption budgets. Also search the label matchers in PromQL rules
  and dashboards.
- A resource name. Search for anything that refers to it: route
  backends, cross-namespace DNS names
  (`<name>.<namespace>.svc.cluster.local`), selectors, and Flux `dependsOn`
  and `healthChecks`.
- A value, a flag, or a CRD field. Search the HelmRelease `values`,
  `valuesFrom`, and `postRenderers`, and the kustomize patches. Include the
  patches that the FluxInstance in `kubernetes/apps/flux-system/flux-instance`
  applies to every HelmRelease, for example a CRD upgrade policy. Also
  include the custom resources that other apps create from the CRDs of the
  dependency (`apiVersion: <group>`).

Some apps keep their configuration on their volume, because you configure
them in their own UI. That configuration is not in git, so a search of this
repository proves nothing about it. The change can affect an integration or
a configuration item in such an app. If it does, say that the review was
not able to make sure of it.

You can tell the author to rename something or to set a new value. Before
you do, make sure that the new name or value is correct. Use the diff of the upstream
pull request, or the chart template at the new tag. Read the template raw
with this command:

```
gh api -H "Accept: application/vnd.github.raw+json" repos/<owner>/<repo>/contents/<path>?ref=<tag>
```

If you cannot make sure, write it as something to look at after the merge,
not as an edit. A wrong edit that someone applies does more damage than no
edit.

Put a finding on the changed line in the diff. In its explanation, name the
file and the line that depend on the change.

## What a check proves

- Write out each inference. Write only an inference that the evidence
  supports. A successful render of the new chart proves only that the tag
  exists and that the values fit its schema.
- A release can change a default to a value that this repository already
  sets explicitly. Then nothing changes here. A prerequisite that comes
  with the release, for example a kernel or Kubernetes version, needs its
  own check.
- Compare a version requirement with the version that this repository
  targets. Read the target from the file that pins it. A target in git does
  not prove that every node runs that version. Say that the line depends on
  the target.
- Put the evidence in the claim: the value that you found and the file that
  it came from, or the source that says so. "`auth.enabled: true` already
  set in the HelmRelease values" carries its proof. "auth checked" does
  not.
- Something that the repository cannot show is not a check. Say what you
  were not able to make sure of, and what to look at after the merge.

## In kritika

The confidence scorer of kritika grades the review by its checked lines and
the sources that it lists. The scorer has the description and the diff, but
no tools. It never sees the commands that the review ran or their output. A
release-note entry that no checked line covers lowers the score.

If the pull request is an update, give each of these items a checked line
of its own:

- Each entry under the breaking changes in the release notes.
- Each entry under the upgrade notes in the release notes.
- Each step that the upgrade guide names.

Do this also for an entry that affects nothing here. Name the entry, then say why
it does not apply or what it changes.

The kritika checkout has no git history, so only the head renders, as
`references/rendering.md` describes.
