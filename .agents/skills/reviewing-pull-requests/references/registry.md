# Reading a registry with crane

If `crane` is available, use it instead of `curl` for a container image or
an OCI chart. It reads registries without credentials and prints JSON.

- `crane ls <registry>/<repo>` lists tags. `sha256-…` tags are signatures
  and attestations, not versions.
- `crane digest <image>:<tag>` gives the digest of a tag.
- `crane manifest <image>:<tag>` shows the platforms of a multi-platform
  image. All nodes of the cluster are `linux/amd64`. If a new tag does not
  include `linux/amd64`, that is a finding.
- `crane config --platform linux/amd64 <image>@<digest>` shows the build
  time of the image (`created`) and its labels.
  `org.opencontainers.image.source` names the upstream project.
  `org.opencontainers.image.revision` names the commit that the image was
  built from. On `ghcr.io/home-operations/*` images, that commit is in
  `home-operations/containers`, not in the repository of the app. The diff
  of that commit shows why a digest-only update was rebuilt.
- `crane config <chart>:<version>` on an `OCIRepository` chart prints its
  `Chart.yaml` as JSON: `version`, `appVersion`, and `sources`. Run it at
  the old tag and at the new tag.
