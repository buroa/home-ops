# Reading a registry

A registry stores container images and OCI charts. A tag is a name for
an image or chart. A digest identifies the exact content.

In kritika, use `fetch_oci`. Set `image` to the registry path with the tag
or digest. If you inspect an OCI chart, omit the `oci://` prefix. If you
list tags, set `image` to the registry path. Set `tags` to the text that
you want to match.

- For an image, make sure that the digest matches the expected value.
  Make sure that the new image supports `linux/amd64`, the platform of
  this cluster. Compare the `created` times and version and revision
  labels for the old and new images. A multi-platform index lists the
  image builds for different platforms. `fetch_oci` shows the
  configuration of the first image in that index. Before you use its
  labels, make sure that they describe the platform you need.
- For the image source, read `org.opencontainers.image.source`. It names
  the project that produces the image. The label
  `org.opencontainers.image.revision` names the build commit. For a
  `ghcr.io/home-operations/*` image, look up that commit in
  `home-operations/containers` to see what changed in the build.
- For a chart, read the chart configuration at both tags. Look for
  `version`, `appVersion`, `sources`, and `home`.

If `crane` is available in a local review, use it to inspect the registry.

- `crane ls '<registry>/<repo>'` lists tags. Skip `sha256-…` signature and
  attestation tags when looking for versions.
- `crane digest '<image>:<tag>'` resolves a tag to its digest.
- `crane manifest '<image>:<tag>'` shows the platforms of a multi-platform image.
- `crane config --platform linux/amd64 '<image>@<digest>'` shows the image
  configuration.
- `crane config '<chart>:<version>'` shows the chart configuration.
