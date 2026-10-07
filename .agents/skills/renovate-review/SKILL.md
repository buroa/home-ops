---
name: renovate-review
description: Use for every pull request on a renovate/ branch in this Flux repository - Helm chart bumps (OCIRepository tags), container image and digest bumps, Talos and Kubernetes upgrades, GitHub Action and Renovate preset bumps. Gives the flate command that renders the change against main, where each dependency's upstream changelog lives, which breaking changes matter here, and how to search this repository for what depends on the old behavior.
---

# Review a Renovate PR

Decide whether the update is safe to merge: what changed upstream, and
whether anything in this repository depends on it. A merge rolls
`kubernetes/` out within minutes.

## 1. Render the change

```
flate diff hr <name> --path kubernetes/flux/cluster --no-progress
```

`<name>` is the changed HelmRelease; omit it to diff everything the PR
touched, which for a shared chart under `kubernetes/flux/repositories/oci/`
is every HelmRelease using it. flate compares against `main` on its own
and renders only the touched subtree, in about a second.

The output is what the cluster will receive. Take renamed labels, resource
names, ports and selectors from it, not from the changelog. Chart and
version labels are ignored, so an empty diff means the new chart renders
the same with this repository's values. CRDs and Secrets are left out. A
failure of another resource or an unreachable source says nothing about
the update; only this HelmRelease failing because of it is a finding.

For an image bump the diff only confirms the tag; the review is the
changelog against the HelmRelease's `env`, `args` and mounted config.

## 2. Read upstream

Start with the PR body's **Release Notes**. Go upstream when they are
missing or truncated, and always for a major, a 0.x minor or a jump over
several releases; read every intermediate release.

- **Chart**: the app's `ocirepository.yaml` names the registry path.
  `oci://<registry>/<owner>/...` is that owner's chart repository on
  GitHub. Multi-chart repositories tag releases `<chart>-<version>`, and
  `gh api repos/<owner>/<repo>/compare/<chart>-<old>...<chart>-<new>`
  lists every template that changed.
  `ghcr.io/home-operations/charts-mirror/<chart>` mirrors a chart
  published outside OCI; `apps/<chart>/metadata.yaml` in
  `home-operations/charts-mirror` names the upstream. When `appVersion`
  moves, read the app's changelog too.
- **Image**: usually `<owner>/<repo>` on GitHub, otherwise the
  `org.opencontainers.image.source` annotation. `ghcr.io/home-operations/<app>`
  is built in `home-operations/containers`: `apps/<app>/docker-bake.hcl`
  holds the upstream version and source, and that path's commit history
  explains a digest-only bump.
- **Talos and Kubernetes** (`tuppr/upgrades/`): tuppr upgrades the live
  nodes after the merge; the `talos/` templates only shape new nodes.
  Check the Talos release notes for the supported Kubernetes versions.
- **"See #123"**: read the PR that closed the issue
  (`gh issue view <n> -R <repo> --json closedByPullRequestsReferences`);
  it shows the real scope.

Beyond `BREAKING` and `!:` markers, flag changed defaults (auth, storage
class, ports, probes), one-way migrations, raised minimum Kubernetes, Flux
or Talos versions, and a label value or resource name that changed while
its key stayed, such as a dropped name prefix: a search for the key still
matches, but selectors on the old value break. Settle "prefix removed"
from the diff or the template, not from the sentence.

## 3. Search for exposure

Ask whether anything here depends on the old thing, not whether this
repository sets it. Search `kubernetes/` for each name and value the diff
removed: `matchLabels` and `selector` in monitors, network policies and
disruption budgets; PromQL label matchers in rules and dashboards; route
backends and `<name>.<namespace>.svc.cluster.local`; Flux `dependsOn` and
`healthChecks`; `values`, `valuesFrom` and `postRenderers`; custom
resources from the dependency's CRDs. For an action, check the `with:`
inputs and outputs used in `.github/workflows/`; for a Renovate preset,
`.renovate/` and `.renovaterc.json5`.

Confirm a prescribed rename or value from the diff or the upstream
template at the new tag:

```
gh api -H "Accept: application/vnd.github.raw+json" "repos/<owner>/<repo>/contents/<path>?ref=<tag>"
```

Unconfirmed, state it as a check in the finding, without a suggestion: a
wrong suggestion that gets applied is worse than none.

## 4. Rules and report

`.renovate/autoMerge.json5` says whether the PR merges itself once checks
pass; this review is then the last look, so say what to watch after the
merge. `.renovate/groups.json5` says which PRs carry several packages,
each to be reviewed. The shared preset puts `!` in the title for majors
and 0.x minors; note a title that disagrees with the `type/*` label.

Anchor each finding to the bumped line and name the dependent file and
line. A breaking change that touches nothing here is not a finding: one
sentence in the summary, with the search that came up empty. ExternalSecret
values are not visible; judge key names and mappings only.
