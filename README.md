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
| `build-image.yml` | Builds the image and pushes it to GHCR as `<yyyy-mm-dd>-<run>-<sha7>`; refuses to overwrite an existing tag; only from the default branch. |
| `image-deps.yml` | Builds one Dockerfile stage (`deps` by default) without pushing, so a broken `bundle install` inside the image fails on the PR. |
| `ghcr-cleanup.yml` | Deletes old build images (`<yyyy-mm-dd>-<run>-<sha7>`, and the earlier `sha-<7>`), keeping the 3 newest and every tag a chart values file still references. |

Every third-party action is pinned to a full commit SHA with its version in a
comment; Dependabot bumps both weekly.

## Calling

A check's name has three levels, "what / where / how", and each side sets
one: the caller's workflow name says what is checked (`CI`, `Image deps`),
the caller's job id says where (`app`, or the component in a matrix), and
the reusable workflow's job name says how — the command that ran
(`bin/ci`, `docker build --target deps`). So the examples below name their
job `app`, and a check reads `CI / app / bin/ci`.

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
  app:
    uses: elon-kogan/actions/.github/workflows/ci.yml@v1
```

A monorepo runs the same workflow once per component:

```yaml
jobs:
  component:
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
  app:
    uses: elon-kogan/actions/.github/workflows/build-image.yml@v1
```

Inputs: `image` (default `ghcr.io/<owner>/<repo>`), `target` (default
`prod`), `platforms` (default `linux/amd64`). The GHCR package must grant the
calling repository write access if it predates the workflow.

The tag is `<yyyy-mm-dd>-<run>-<sha7>`, e.g. `2026-10-22-42-98d8fd8`: the UTC
build date, the caller's `github.run_number` as is, and the first seven hex
characters of the commit. It is the only tag pushed, and the one shape ArgoCD
Image Updater accepts (`^\d{4}-\d{2}-\d{2}-\d+-[0-9a-f]{7}$`).

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
  app:
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
  app:
    uses: elon-kogan/actions/.github/workflows/ghcr-cleanup.yml@v1
    with:
      dry-run: ${{ inputs.dry-run || false }}
```

Inputs: `package` (default: the repository name), `keep-tags-from` (globs of
chart values files whose `image.tag` is kept; default
`deploy/chart/values-*.yaml`), `delete-tags` (tags outside the build shapes
to delete as well, `?` and `*` wildcards allowed; default none), `dry-run`
(default `false`). Run it once with
`dry-run: true` before letting the schedule delete anything. The GHCR package
must grant the calling repository the Admin role under "Manage Actions
access": write is enough to push an image, not to delete one.

## Versions

Two kinds of tag. `vX.Y.Z` is exact and never moves: a release is the
commit it was cut on. `vX` is what callers reference (`@v1`) and is
re-pointed at every compatible release, so callers pick up fixes and action
bumps without a change on their side. There are no moving minor tags.

Which part to raise:

- patch: fixes and pinned-action bumps, nothing a caller sees changes.
- minor: new behaviour behind the same inputs. The image tag moving from
  `sha-<7>` to `<yyyy-mm-dd>-<run>-<sha7>` is the model case: nothing a
  caller writes changed, what it gets did.
- major: a caller has to change something — a renamed input, a changed
  default, a removed workflow. One such commit makes the whole release
  major. It ships as `vX+1`; the old `vX` stays where it was, and each
  caller moves its `uses:` line when it is ready. Versions only go up, so
  an old major gets no patch once the next one is out.

`./release vX.Y.Z` is the only way a tag is made here. It refuses a dirty
tree, a branch other than main, a HEAD that is not origin/main, an existing
tag and a number that is not above the highest release; then it creates the
annotated exact tag, re-points the major, pushes both (the major with a force
limited to that one ref) and creates the GitHub release with generated notes.
Run again after a partial failure, it keeps an exact tag already on HEAD and
finishes the steps after it. `/release` in Claude Code reads the log since
the last release, proposes the level with the reason per commit, and runs the
line only on a yes.

A reusable workflow cannot run on its own; `actionlint` (with shellcheck),
run by `lint.yml` on every pull request and locally, is the only check here.
A change is proven by a caller referencing it at `@main` before the tag
moves: the script checks that main is merged and pushed, not that a caller
ran it.
