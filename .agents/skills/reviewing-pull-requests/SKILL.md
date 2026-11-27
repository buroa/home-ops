---
name: reviewing-pull-requests
description: Review home-ops pull requests and Renovate updates. Find what changed in other projects and what those changes affect under kubernetes/.
---

# Reviewing pull requests in home-ops

Flux applies `kubernetes/` from `main` to the cluster. A merge goes live
within minutes. Secrets are ExternalSecrets that come from 1Password.
Judge their key names and mappings from the manifests.

A HelmRelease tells Flux which chart and values to use.

## A Renovate update

A dependency is a package that this repository uses. Upstream is the
project that publishes it. If the pull request uses a `renovate/`
branch, find what changed upstream between the two versions. Find what
depends on those changes in this repository.

- Read GitHub with `gh`.
- Read the changelog or the documentation site of a project with `curl`.
- Inspect images and OCI charts with `fetch_oci` in kritika or `crane`
  locally. See `references/registry.md`.

### The update

The diff shows the lines that the pull request changes. Get these facts
from the title, body, labels, and diff.

- The package and its type: container image, Helm chart as an
  `OCIRepository`, GitHub Action, mise tool, Grafana dashboard, or Renovate
  preset.
- The old and the new version or digest. A digest identifies the exact
  content of an image or chart.
- The update type, from the `type/*` label.
- The changed files. These are the places that use the dependency.

Read the collapsed `Release Notes` section in the pull request first.
Use the upstream release notes in these cases.

- The section is missing, cut short, or not clear.
- The update is a major update.
- The update skips several versions. Read the notes for each release after
  the old version up to and including the new version. A breaking change
  can occur in a release that the update skips.

### Finding the upstream

- An image at `<registry>/<owner>/<repo>` usually comes from
  `<owner>/<repo>` on GitHub. If it does not, the pull request body or the
  `org.opencontainers.image.source` label of the image names the source.
  Read the label as `references/registry.md` describes.
- Inspect the pinned image digest. Compare its version and revision labels
  with the release under review.
- Find the chart source in the `OCIRepository` chart template of the app
  (`spec.chart` in `helmrelease.yaml`). The `spec.chart.spec.url` gives the
  registry path. The `spec.chart.spec.ref.tag` gives the chart version.
- For a chart at `oci://<registry>/<owner>/...`, read `sources` or `home`
  from the chart configuration in the registry. These fields can name
  the GitHub repository.
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
  chart name, for example `<chart>-1.2.3` in place of `v1.2.3`. If you
  cannot find a release, list the releases. Look for the chart name.
- In a digest-only update, the version tag stays the same. Compare
  `created` and `revision` for the old and new digests. Say whether the
  project rebuilt the image. If you can find the reason, give it. A
  security fix or a `latest`-style tag can explain a rebuild. Name the
  source revision of the new build.
- If a release note says "see #2785", read the reference with
  `gh issue view` or `gh pr view`. If it is an issue, find the merged pull
  request that closed it
  (`gh pr list -R <repo> --search "#<n>" --state merged`). The pull request
  shows what changed and the scope of the change. The issue and the release
  note sometimes do not.

### What breaks

A CRD defines a Kubernetes resource type.

Flag these items in each release:

- `BREAKING CHANGE`, `⚠`, or `!:` markers.
- Helm values, CRD fields, environment variables, and flags that the
  release removes or renames.
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
"Chart name prefix removed" can refer to label values or resource names.
Use the pull request or the template to find which one changed.

Look for image changes outside the diff. A chart or operator can change
its default image or let another operator choose the image. Report the
old and new image versions. Read the chart or operator source at both
versions. Make sure that you know whether the HelmRelease pins the image.

If the update is under `kubernetes/`, follow
[A change under `kubernetes/`](#a-change-under-kubernetes). If a breaking
change has no dependents here, say what you searched in the review
summary.

### Renovate's rules

`.renovaterc.json5` defines how Renovate treats this update:

- Its `automerge` rules say whether the pull request merges without a
  person. If it does, the review is the last look before the merge.
- Its `groupName` rules say which pull requests update several packages
  together. The review must cover each of these packages.
- Its labels, and the `!` that the shared preset adds to a major update,
  must agree with the title. If they do not agree, add a note.

## A change under `kubernetes/`

If flate is available and a chart version or HelmRelease value changes,
follow `references/rendering.md` to render the HelmRelease.

### What depends on the change?

Search `kubernetes/` for uses of the old name, label, value, or field.

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

Some apps save configuration from their UI on a volume. If a change can
affect that configuration or a UI integration, say what Git cannot show.

Before you suggest a new name or value, make sure that it is correct.
Read the upstream diff or the chart template at the new tag. Use this
command to get the template.

```
gh api -H "Accept: application/vnd.github.raw+json" repos/<owner>/<repo>/contents/<path>?ref=<tag>
```

If you cannot make sure that the edit is correct, describe what to examine
after the merge. An incorrect edit can cause a new problem.

Put a finding on the changed line in the diff. In its explanation, name the
file and the line that depend on the change.

## What the evidence shows

- Say what each check showed. A successful render shows that flate
  fetched the chart and used the values of this repository.
- A release can change a default to a value that this repository already
  sets explicitly. Then nothing changes here. A prerequisite that comes
  with the release, for example a kernel or Kubernetes version, needs its
  own check.
- Compare a version requirement with the version that this repository
  targets. Read the target from the file that pins it. A target in git does
  not prove that every node runs that version. Say that the line depends on
  the target.
- Give the value that you found. Name the file or source that shows it.
  For example, write "`auth.enabled: true` already set in the HelmRelease
  values".
- Say what the repository cannot show. Name what to examine after the
  merge.

## In kritika

The confidence scorer rates the review. It sees the review and its
sources. It sees which commands ran or failed. It relies on the review
for what the command output showed.

For a dependency update, give each breaking change, upgrade note, and
upgrade step its own checked line or finding. State what you examined.
State what you found. Name the source. If an item does not apply here,
say why.

Use the explicit baseline in `references/rendering.md` to diff a
HelmRelease from the kritika checkout.
