# Building a custom container image

The [Advanced usage] page covers the *light* path for customizing a container: pick a
base image and add packages to it through the base image's dependency manifests. That
path is enough for most workflows and should always be your first choice.

Some workflows need more — for example a binary that has to be **compiled from source**,
or an R/Python package installed from a **specific GitHub branch**, or a **custom entry
point**. None of those can be expressed by simply listing packages. This page shows the
recommended way to build a fully custom image for cases like these, using the
**GLM-AED-FLARE** image (used to run the [FLARE](https://github.com/FLARE-forecast) lake
water-quality forecast) as a running example.

!!! note
    This page describes a *recommended* structure. The FLARE image that ships in
    FaaSr-Docker today works correctly but is built slightly differently (it does
    everything in a single Dockerfile); the differences, and why the layered approach
    below is preferable for new images, are summarized in
    [How the shipped FLARE image differs](#how-the-shipped-flare-image-differs).

## The guiding principle

A FaaSr image is built in two layers (see [Building containers]):

- a **base image** — a language runtime plus common dependencies, built with the
  `py-base` / `rocker-base` build actions;
- a **platform-specific image** — built *from* a base image, adding the code needed to
  run on a provider (GitHub Actions, AWS Lambda, GCP, …).

The principle for a custom image is simple:

> **Put as much as possible into a customized base image. Drop down to a dedicated
> Dockerfile only for the things a package list cannot express.**

Everything you put in the base is declared once, cached, and reused across rebuilds and
across every image that builds from it. A dedicated Dockerfile is harder to maintain, so
you want it to carry only what genuinely needs it.

### Decide what goes where

| If you need to… | Put it in… |
| --- | --- |
| Start from a specialized stack (e.g. geospatial) | the base image — set `BUILD_FROM` when building it |
| Install system libraries (`apt`) | the base image — `apt-packages.txt` |
| Install CRAN / PyPI / GitHub-released packages | the base image — `R_packages.R` / `requirements.txt` |
| Compile a binary from source | a dedicated Dockerfile (multi-stage) |
| Install a package from a specific Git **branch/commit** chosen at build time | a dedicated Dockerfile (parameterized `ARG`) |
| Ship a custom entry point | a dedicated Dockerfile |

## Recommended path, with FLARE as the example

### Step 1 — Build a base image that matches your science

The FLARE forecast's first hard dependency is the R package **`ncdf4`**, which compiles
against the NetCDF development headers (`libnetcdf-dev`); it also pulls in `udunits2`,
`CFtime`, and friends. The default FaaSr R base is built from `rocker/tidyverse`, which
does **not** include those system libraries — so `ncdf4` would fail to build on it.

`rocker/geospatial` already provides GDAL, GEOS, PROJ, **`libnetcdf-dev`**, and udunits,
plus prebuilt spatial R packages. So the right move is to build a FaaSr base **from**
geospatial. When you run the `rocker-base` build action (see [Building containers]):

- set **`BUILD_FROM`** to `rocker/geospatial:4.4.2`
- set **`BASE_NAME`** to something descriptive, e.g. `base-flare`

This produces a base that has the FaaSr runtime *and* the geospatial system stack. Add
the forecast's CRAN packages (`rLakeAnalyzer`, `tidymodels`, `arrow`, `rstac`,
`aws.s3`, …) to the base's `R_packages.R` so they are baked in and reused.

!!! tip
    The `rocker-base` action's own input documentation already lists `base-geospatial`
    and `base-flare` as example base names — building a geospatial-derived base is an
    anticipated, supported pattern, not a workaround.

### Step 2 — Add a dedicated Dockerfile for what the base can't carry

Three things in the FLARE image cannot be expressed as a package list, so they belong in
a dedicated platform Dockerfile that builds **`FROM` your `base-flare` image**:

**Compile GLM from source (multi-stage).** GLM (the hydrodynamic engine FLARE drives)
has no reliable prebuilt binary for every runner architecture, so it is compiled in a
throwaway build stage and only the resulting binary is copied into the runtime image:

```dockerfile
# --- build stage: compile GLM, kept out of the final image ---
FROM <your-base-flare-image> AS glm_builder
RUN apt-get update && apt-get install -y \
    git build-essential gfortran libnetcdf-dev libgd-dev libxml2-dev m4 fakeroot debhelper
WORKDIR /build
RUN git clone --depth 1 https://github.com/AquaticEcoDynamics/AED_Tools.git \
    && cd AED_Tools && ./fetch_sources.sh glm \
    && cd GLM && git fetch origin && git switch v4alpha && cd .. \
    && ./clean.sh && ./build_glm.sh --no-gui

# --- runtime stage ---
FROM <your-base-flare-image>
COPY --from=glm_builder /build/AED_Tools/GLM/glm /opt/glm/glm
RUN chmod +x /opt/glm/glm
ENV GLM_PATH=/opt/glm/glm
```

**Install the model package from a build-time-chosen branch.** Exposing the install
repo and version as build arguments lets the same Dockerfile build against the canonical
FLAREr repository *or* against your own fork and development branch, without edits:

```dockerfile
ARG FLARER_INSTALL_REPO
ARG FLARER_VERSION
RUN Rscript -e "library(remotes); install_github(paste0('${FLARER_INSTALL_REPO}', '@', '${FLARER_VERSION}'), dependencies = TRUE)"
```

**Ship a custom entry point** when your function needs setup the standard entry point
does not perform. FLARE needs one specific thing: its R code resolves paths **relative to
the repository root** (where the `configuration/` directory lives), but FaaSr extracts a
function's GitHub repository into a tarball folder (e.g. `Owner-Repo-<sha>`), so the
working directory at startup is not the repo root. The FLARE entry point therefore locates
the configuration directory and changes the working directory to the repo root *before*
the function runs:

```python
def setup_flare_configuration_directory():
    # find the configuration/ directory, then chdir to the repo root above it
    # so FLARE's relative paths (config, observations) resolve correctly
    ...
    os.chdir(repo_root)
```

It is then installed exactly as any platform image installs its entry point:

```dockerfile
ENV FAASR_PLATFORM="github"
RUN mkdir -p /action
COPY faasr_entry_flare.py /action/faasr_entry.py
WORKDIR /action
CMD ["python3", "faasr_entry.py"]
```

!!! tip "Keep a custom entry point a minimal delta"
    A custom entry point should add only your function-specific logic (here, the working-
    directory setup) on top of the standard `faasr_entry.py`. Resist folding in unrelated
    fixes: if you find a general improvement while editing your copy, contribute it back to
    the standard entry point instead. Otherwise your fork and the standard file drift apart
    and become hard to keep in sync.

Because the geospatial stack and the CRAN packages already live in `base-flare`, this
Dockerfile stays small: it only compiles GLM, installs the model package from the branch
you point it at, and installs the entry point.

!!! tip "Avoid the GitHub API rate limit"
    Steps that call `install_github` hit the GitHub API, which is limited to 60 requests
    per hour for anonymous access — enough to fail a multi-package install. Pass a token
    through a `GITHUB_PAT` build argument (the FLARE build wires in the workflow's
    built-in `GITHUB_TOKEN`) to raise the limit and make the build reliable.

### Step 3 — Build and publish the image

Give your custom Dockerfile its own `workflow_dispatch` GitHub Actions workflow whose
inputs map onto its build arguments, so you can rebuild variants from the Actions tab.
The FLARE build (`build_github_actions_glm_aed_flare_rs.yml`) exposes:

| Input | Example | Purpose |
| --- | --- | --- |
| `BASE_IMAGE` | `rocker/geospatial:4.4.2` *(or your `base-flare`)* | base for the build |
| `TARGET_NAME` | `github-actions-glm-aed-flare-rs` | image name |
| `FAASR_VERSION` | `2.0.5` | FaaSr-py version |
| `FLARER_INSTALL_REPO` | `Ashish-Ramrakhiani/FLAREr` | repo to install the model from |
| `FLARER_VERSION` | `flare-io-on-netcdf-v2` | model branch / tag / commit |
| `GHCR_IO_REPO` | *(your namespace)* | GHCR namespace to push to |

The workflow logs in to GHCR with the built-in `GITHUB_TOKEN`, builds the image, and
pushes it with both a version tag and `latest`:

```text
ghcr.io/<your-namespace>/github-actions-glm-aed-flare-rs:2.0.5
ghcr.io/<your-namespace>/github-actions-glm-aed-flare-rs:latest
```

### Step 4 — Use the image in a workflow

Reference the published image per-action in your workflow JSON under `ActionContainers`:

```json
"ActionContainers": {
    "run_forecast": "ghcr.io/<your-namespace>/github-actions-glm-aed-flare-rs:latest"
}
```

You can set this in the [FaaSr Workflow Builder Web UI] under **Action Containers** (see
[Creating and editing workflows]). Because this is a non-default container, when you
[register the workflow] with `FAASR REGISTER` you must tick the **Allow custom
containers** checkbox.

!!! warning
    Only run custom containers that you build yourself or whose provenance you trust.
    Review the [security best practices] before using custom containers.

## Reusable techniques

The FLARE image demonstrates patterns that apply to any custom scientific-computing
image:

- **Match the base to the science** — start from a base such as `rocker/geospatial` that
  already carries the heavy system libraries your packages compile against, and build
  your FaaSr base from it.
- **Multi-stage compile-from-source** — build a binary in a throwaway stage and copy only
  the result into the runtime image, keeping the toolchain out of the published image.
- **Parameterized installs** — expose install repositories and versions as build
  arguments so one Dockerfile builds multiple variants (canonical vs. your fork).
- **A token for `install_github`** — pass `GITHUB_PAT` to avoid the anonymous API rate
  limit during the build.
- **A custom entry point** — ship your own `faasr_entry.py` when the function needs
  pre-execution setup the standard entry does not do (most commonly setting the working
  directory to the repo root so relative paths resolve), and keep it a minimal delta over
  the standard entry.

## How the shipped FLARE image differs

The `glm-aed-flare-rs` image currently in FaaSr-Docker works correctly and does not need
to change. It differs from the recommended path above in one way: it does everything in a
**single Dockerfile built directly from `rocker/geospatial`**, rather than first building
a reusable `base-flare` image. As a result it installs the FaaSr runtime and the full
CRAN package list inside that one Dockerfile (using a `glm_aed_flare_rs_packages.txt`
manifest installed with a single `install.packages` call, and a
`--break-system-packages` pip install to satisfy PEP 668 on Ubuntu 24.04).

Its custom entry point also carries a couple of general (non-FLARE) tweaks that were
edited directly into the forked copy; following the "minimal delta" guidance above, such
changes are better contributed back to the standard entry point so the two do not drift.

That is a perfectly valid way to produce a working image. For **new** custom images,
prefer the layered approach: it pushes the reusable environment (geospatial stack + CRAN
packages) into a base image that is cached and shared, and keeps the dedicated Dockerfile
small and focused on the parts that truly require it.

[Advanced usage]: advanced.md
[Building containers]: docker_build.md
[FaaSr Workflow Builder Web UI]: workflows.md
[Creating and editing workflows]: workflows.md
[register the workflow]: register_workflow.md
[security best practices]: security.md
