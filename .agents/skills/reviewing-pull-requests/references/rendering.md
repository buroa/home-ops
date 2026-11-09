# Rendering a HelmRelease with flate

If a chart version or the values of a HelmRelease change, render the
HelmRelease with the values of this repository. Run flate from the
repository root, and select the namespace of the HelmRelease with `-n`.

If git history is available, diff the render against the base. flate
renders the base from that revision:

```
flate diff hr <name> -n <namespace> --base origin/main --no-progress -o diff
```

If the render of a HelmRelease is the same on both sides, flate leaves it
out of the diff. A diff that names it then exits nonzero with
`no HelmRelease named "<name>"`. This error means that the change does not
reach the output of the HelmRelease. It does not mean that the HelmRelease
is missing.

If no history is available, as in the kritika checkout, build only the
head:

```
flate build hr <name> -n <namespace> --no-progress
```

flate reports every failure in the namespace, not only the failure of the
HelmRelease that you asked for. If any failure occurs, it exits nonzero.
Two conditions make a failure a finding on the changed line: the failure
belongs to the HelmRelease under review, and the change caused it.
Examples are a value that the chart schema rejects, or a template that
fails with the values of this repository. A failure of another HelmRelease
tells you nothing about the change. A source that flate was not able to
fetch also tells you nothing.

A successful render is an offline approximation of what the cluster
applies. It includes the patches that Flux applies from a parent
Kustomization. It has these limits:

- It leaves out CRDs and Secrets.
- Templates that branch on Kubernetes capabilities see a bundled version.
  This is the Kubernetes version of flate, not the version of the cluster.

Within these limits, get the label values, resource names, and ports for
the search for dependents from the render. Do not get them from your
reading of the template. If the whole output is too long, narrow it to the
kinds that you need with `--only-kinds <Kind>,<Kind>`. Sometimes only the
head renders. Then find what the base produced from upstream. You can also
find it from the names that this repository already refers to.
