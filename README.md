# umatix actions

GitHub Actions steps for customers of the umatix platform. The steps run on the runners umatix provides for a
project; the registry and its credentials come from the runner, so nothing secret is in this repository.

## Runners

A project with CI gets a runner scale set named after the project (`runs-on: <project>`). Jobs without a
`container:` run in the runner itself; jobs with `container:` or `services:` get a pod of their own, which reaches
DNS and the internet only (no buildkitd, no registry), so image builds belong in jobs without a container. Every
runner provides:

| Variable | What |
|---|---|
| `BUILDKIT_HOST` | The project's own rootless buildkitd (no Docker daemon anywhere) |
| `UMATIX_REGISTRY` | The project's prefix in the registry; workloads run only images from under it |
| `DOCKER_CONFIG` | The push credential for that prefix: read and create, never overwrite or delete |
| `SSL_CERT_DIR` | The CAs the registry's certificate needs |

## `build`

Builds an image with BuildKit, scans it with Trivy (a critical vulnerability that has a fix fails the step), and
pushes it as `$UMATIX_REGISTRY/<component>:sha-<revision>`. Tags are write-once: a tag that exists is kept, and the
step reports its digest.

```yaml
jobs:
  images:
    runs-on: my-project
    steps:
      - uses: actions/checkout@v7
      - id: api
        uses: umatix/actions/build@v1.0.0
        with:
          component: api
          dockerfile: services/api/Dockerfile
          context: .
          target: production
          build-args: |
            VERSION=1.2.3
          push: ${{ github.ref == 'refs/heads/main' }}
      - run: echo "${{ steps.api.outputs.image }}@${{ steps.api.outputs.digest }}"
```

| Input | Default | |
|---|---|---|
| `component` | (required) | Image name under the project's prefix |
| `context` | `.` | Build context, relative to the workspace |
| `dockerfile` | `Dockerfile` | Relative to the workspace |
| `target` | last stage | Build stage |
| `build-args` | none | One `KEY=value` per line |
| `revision` | `github.sha` | Commit the image is built from and tagged with; set it under `workflow_run` |
| `push` | `false` | `true` pushes after a clean scan; anything else builds and scans only |
| `ignore-file` | none | A Trivy ignore file for this image only (see below) |

Outputs: `image` (the pushed reference, empty without push), `tag` (`sha-<revision>`), `digest`.

**Exceptions.** A critical vulnerability with a fix fails the step. When one cannot be fixed yet (an upstream that
no longer ships fixed images), list it in an ignore file and pass it with `ignore-file` for that one image: Trivy's
`.trivyignore.yaml` lets every entry name the affected paths, a statement why it does not apply, and an expiry
(`expired_at`): from that date, 00:00 UTC, the gate fails again. Neither a `.trivyignore` nor a `trivy.yaml` in the
repository is read on its own, so no exception reaches an image by accident.

The images are `linux/amd64`. Releases are immutable tags (`v1.0.0`, `v1.1.0`, …); pin one, or a commit.

## Deploying

The platform deploys a project from a path in the project's own repository: merging a change there is the
deployment. So a release lane builds the images with `build` and then changes the image tags in that path, through
a pull request or a commit of its own.
