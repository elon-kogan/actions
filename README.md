# actions

Reusable GitHub Actions workflows shared by my app repositories. Each file
under `.github/workflows/` is one `workflow_call` workflow; an app calls it
with a single `uses:` line instead of carrying its own copy. The exception is
`lint.yml`, which lints this repository itself.

What stays in the app: the trigger and its paths filter, `permissions`,
`concurrency` (`ghcr-cleanup.yml` is the one exception: it serialises itself
per package, because the cleanup action is not safe in parallel), and every
secret. What lives here: the steps, the pinned action versions, and the
reasons for them.

| Workflow | Does |
| --- | --- |
| `ci.yml` | Checks out the app, installs the Ruby from its `.ruby-version` and the gems, runs one command (`bin/ci` by default). |
| `build-image.yml` | Builds the image and pushes it to GHCR as `sha-<7>`; refuses to overwrite an existing tag; only from the default branch. |
| `image-deps.yml` | Builds one Dockerfile stage (`deps` by default) without pushing, so a broken `bundle install` inside the image fails on the PR. |
| `ghcr-cleanup.yml` | Deletes old `sha-*` images, keeping the 3 newest and every tag a chart values file still references. |

Every third-party action is pinned to a full commit SHA with its version in a
comment; Dependabot bumps both weekly.

## Calling

### CI

```yaml
name: CI

on:
  pull_request:
    paths-ignore:
      - deploy/**
      - docs/**
      - "**.md"
  push:
    branches: [main]
    paths-ignore:
      - deploy/**
      - docs/**
      - "**.md"

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: elon-kogan/actions/.github/workflows/ci.yml@v1
```

A monorepo runs the same workflow once per component:

```yaml
jobs:
  ci:
    strategy:
      matrix:
        component: [contract, engine, plugin, templates]
    uses: elon-kogan/actions/.github/workflows/ci.yml@v1
    with:
      working-directory: components/${{ matrix.component }}
      command: bundle exec rake test
```

Each component directory needs its own `.ruby-version`; the workflow does
not take a Ruby version, so the app is the only place it is written.

### Build image

```yaml
name: Build image

on:
  push:
    branches: [main]
    paths-ignore:
      - deploy/**
      - docs/**
      - "**.md"
      - .github/workflows/ci.yml
      - .github/workflows/image-deps.yml
  workflow_dispatch:

permissions:
  contents: read
  packages: write

concurrency:
  group: build-${{ github.sha }}
  cancel-in-progress: false

jobs:
  build:
    uses: elon-kogan/actions/.github/workflows/build-image.yml@v1
```

Inputs: `image` (default `ghcr.io/<owner>/<repo>`), `target` (default
`prod`), `platforms` (default `linux/amd64`). The GHCR package must grant the
calling repository write access if it predates the workflow.

### Image deps

```yaml
name: Image deps

on:
  pull_request:
    paths:
      - Dockerfile
      - .dockerignore
      - Gemfile
      - Gemfile.lock
      - .ruby-version

permissions:
  contents: read

concurrency:
  group: image-deps-${{ github.ref }}
  cancel-in-progress: true

jobs:
  deps:
    uses: elon-kogan/actions/.github/workflows/image-deps.yml@v1
```

Inputs: `target` (default `deps`), `platforms` (default `linux/amd64`).

### GHCR cleanup

```yaml
name: GHCR cleanup

on:
  schedule:
    - cron: "17 4 * * 1"
  workflow_dispatch:
    inputs:
      dry-run:
        type: boolean
        default: true

permissions:
  contents: read
  packages: write

jobs:
  cleanup:
    uses: elon-kogan/actions/.github/workflows/ghcr-cleanup.yml@v1
    with:
      dry-run: ${{ inputs.dry-run || false }}
```

Inputs: `package` (default: the repository name), `keep-tags-from` (globs of
chart values files whose `image.tag` is kept; default
`deploy/chart/values-*.yaml`), `delete-tags` (tags outside `sha-*` to delete
as well; default none), `dry-run` (default `false`). Run it once with
`dry-run: true` before letting the schedule delete anything. The GHCR package
must grant the calling repository the Admin role under "Manage Actions
access": write is enough to push an image, not to delete one.

## Versions

Callers reference `@v1`. The tag is created after the first merge and moved
onto every later release that keeps the inputs and behaviour compatible, so
callers pick up fixes and action bumps without a change on their side. A
change that breaks a caller — a renamed input, a changed default, a removed
workflow — ships as `v2`, and each caller moves when it is ready.

A reusable workflow cannot run on its own; `actionlint` (with shellcheck),
run by `lint.yml` on every pull request and locally, is the only check here.
A change is proven by a caller referencing it at `@main` before the tag
moves.
