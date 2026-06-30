# Building a fully custom image: the FLARE example

The [Advanced usage] page describes the *light* path for customizing a container:
add packages to a base image, optionally with small edits to a platform-specific
Dockerfile. That path keeps the standard FaaSr build structure intact and is the
right choice for most workflows.

Some workflows need more than that. When a function depends on a binary that has
to be **compiled from source**, on a **non-FaaSr base image**, or on a build that
**adds its own stages and steps**, you write a dedicated Dockerfile and a dedicated
build workflow instead of editing the stock ones. This page walks through a complete,
real example that does all three: the **GLM-AED-FLARE** image used to run the
[FLARE](https://github.com/FLARE-forecast) lake water-quality forecast as a FaaSr
action.

!!! note
    This is a *worked example*, not a different product. It uses the same
    [FaaSr-Docker repository](https://github.com/FaaSr/FaaSr-Docker) and the same
    GitHub Actions build mechanism described in [Building containers] — it just adds
    its own Dockerfile and workflow rather than reusing the generic ones.

## When you need a fully custom image

Reach for a dedicated Dockerfile (rather than the base-image customization in
[Advanced usage]) when:

- A dependency must be **compiled from source** for the runner architecture — there
  is no installable package or reliable prebuilt binary (here: the GLM hydrodynamic
  engine).
- You want to build **from a specialized base image** rather than a FaaSr base — for
  example `rocker/geospatial`, which already carries the geospatial system libraries
  and R packages the forecast needs.
- Your build needs **extra stages or steps** — a separate compile stage, copying a
  built artifact into the runtime image, a custom entry point, and so on.
- You want to **parameterize what gets installed** (e.g. install your model package
  from a configurable repository and branch) so the same Dockerfile can build several
  variants.

If none of these apply, prefer adding packages to a base image as described in
[Advanced usage] — it is simpler to maintain.

## Anatomy of the FLARE image

The example lives entirely in the [FaaSr-Docker repository](https://github.com/FaaSr/FaaSr-Docker)
and is made of three files:

| File | Role |
| --- | --- |
| `faas_specific/github-actions-glm-aed-flare-rs.Dockerfile` | The custom multi-stage Dockerfile |
| `faas_specific/glm_aed_flare_rs_packages.txt` | The list of R packages baked into the image |
| `.github/workflows/build_github_actions_glm_aed_flare_rs.yml` | The GitHub Actions workflow that builds and publishes the image to GHCR |

To create your own custom image, you add a parallel set of three files (your
Dockerfile, your package manifest, your build workflow) to your fork of FaaSr-Docker.

## The Dockerfile, step by step

The Dockerfile uses a **multi-stage build**: the first stage compiles GLM, the second
stage is the actual runtime image and copies in only the compiled binary. This keeps
the build toolchain (compilers, dev headers) out of the final image.

### Stage 1 — compile GLM from source

```dockerfile
ARG BASE_IMAGE=rocker/geospatial:4.4.2

# --- Stage 1: build GLM (v4alpha) from source for the runner architecture ---
FROM $BASE_IMAGE AS glm_builder
RUN apt-get update && apt-get install -y \
    git build-essential gfortran libnetcdf-dev libgd-dev libxml2-dev \
    m4 fakeroot debhelper \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /build
RUN git clone --depth 1 https://github.com/AquaticEcoDynamics/AED_Tools.git \
    && cd AED_Tools \
    && ./fetch_sources.sh glm \
    && cd GLM && git fetch origin && git switch v4alpha && cd .. \
    && ./clean.sh \
    && ./build_glm.sh --no-gui
```

GLM (General Lake Model) is the hydrodynamic engine FLARE drives. A prebuilt binary is
not reliably available for every runner architecture, so the image compiles `v4alpha`
from source using the AquaticEcoDynamics build scripts. Everything this stage installs
(`build-essential`, `gfortran`, the NetCDF/GD dev headers, `fakeroot`/`debhelper`) is
needed only to compile — none of it leaks into the runtime image.

### Stage 2 — the runtime image

The second stage starts fresh from the same base and declares the build arguments that
parameterize the image:

```dockerfile
FROM $BASE_IMAGE

ARG FAASR_VERSION          # FaaSr-py version to install
ARG FAASR_INSTALL_REPO     # GitHub repo to install FaaSr from
ARG FLARER_INSTALL_REPO    # GitHub repo to install FLAREr from
ARG FLARER_VERSION         # FLAREr branch / tag / commit
ARG GITHUB_PAT             # token for install_github (see rate-limit note)
ENV GITHUB_PAT=${GITHUB_PAT}
```

Parameterizing `FLARER_INSTALL_REPO` and `FLARER_VERSION` means the same Dockerfile can
build the image against the canonical FLAREr repository **or** against your own fork and
development branch — you choose at build time without editing the Dockerfile.

It then installs the runtime system libraries (only the *runtime* NetCDF/GD/gfortran
libraries, not the dev headers) and copies the GLM binary out of stage 1:

```dockerfile
RUN apt-get update && apt-get install -y \
    python3 python3-pip libgd3 libgd-dev libnetcdf19t64 libgfortran5 curl \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy the GLM binary compiled in stage 1
COPY --from=glm_builder /build/AED_Tools/GLM/glm /opt/glm/glm
RUN chmod +x /opt/glm/glm
ENV GLM_PATH=/opt/glm/glm
```

`GLM_PATH` tells the forecast code where to find the binary at runtime.

Next it installs the FaaSr runtime and the R packages:

```dockerfile
# Ubuntu 24.04's Python 3.12 enforces PEP 668; the override is safe inside a
# container because this is the only Python environment.
RUN pip3 install --no-cache-dir --break-system-packages \
    "git+https://github.com/${FAASR_INSTALL_REPO}.git@${FAASR_VERSION}"

COPY glm_aed_flare_rs_packages.txt /tmp/required_packages.txt
RUN Rscript -e "packages <- readLines('/tmp/required_packages.txt'); install.packages(packages, dependencies = TRUE)"

RUN Rscript -e "library(remotes); install_github(paste0('${FLARER_INSTALL_REPO}', '@', '${FLARER_VERSION}'), dependencies = TRUE)"
```

FaaSr-py is installed with pip from a Git URL built out of the `FAASR_INSTALL_REPO` and
`FAASR_VERSION` arguments. The `--break-system-packages` flag is required because the
base image runs Ubuntu 24.04, whose Python enforces PEP 668; it is safe here because the
container has a single, dedicated Python environment. The R packages come from the
manifest file (below), and FLAREr is installed from the configurable repo and branch.

The forecast also depends on the ecological-forecasting ecosystem, installed from GitHub:

```dockerfile
RUN Rscript -e "library(remotes); install_github('eco4cast/neon4cast', dependencies = TRUE)"
RUN Rscript -e "library(remotes); install_github('eco4cast/score4cast', dependencies = TRUE)"
RUN Rscript -e "library(remotes); install_github('eco4cast/read4cast', dependencies = TRUE)"
RUN Rscript -e "library(remotes); install_github('LTREB-reservoirs/vera4castHelpers', dependencies = TRUE)"
```

Finally it sets the platform and installs the entry point:

```dockerfile
ENV FAASR_PLATFORM="github"
RUN mkdir -p /action
COPY faasr_entry_flare.py /action/faasr_entry.py
WORKDIR /action
CMD ["python3", "faasr_entry.py"]
```

The image ships a FLARE-specific entry point (`faasr_entry_flare.py`) copied to the
location FaaSr expects (`/action/faasr_entry.py`).

!!! tip "Avoiding the GitHub API rate limit"
    Several steps call `install_github`. Anonymous GitHub API access is limited to 60
    requests per hour, which a multi-package install can exhaust, causing the build to
    fail. Passing a token through the `GITHUB_PAT` build argument (the build workflow
    wires in the built-in `GITHUB_TOKEN`) raises the limit and makes the build reliable.

## The package manifest

The R packages baked into the image are listed one per line in
`faas_specific/glm_aed_flare_rs_packages.txt` and installed in a single `install.packages`
call. Keeping the list in a separate file makes it easy to add or remove packages without
touching the Dockerfile. An excerpt:

```text
contentid
rLakeAnalyzer
tidymodels
arrow
RcppRoll
rstac
aws.s3
zip
...
```

## The build workflow

The image is built and published by its own GitHub Actions workflow,
`.github/workflows/build_github_actions_glm_aed_flare_rs.yml`, triggered manually with
`workflow_dispatch`. Its inputs map directly onto the Dockerfile's build arguments:

| Input | Default | Purpose |
| --- | --- | --- |
| `BASE_IMAGE` | `rocker/geospatial:4.4.2` | base image for both build stages |
| `TARGET_NAME` | `github-actions-glm-aed-flare-rs` | name of the image to build |
| `FAASR_VERSION` | `2.0.5` | FaaSr-py version tag |
| `FAASR_INSTALL_REPO` | `FaaSr/FaaSr-Backend` | repo to install FaaSr-py from |
| `FLARER_INSTALL_REPO` | `Ashish-Ramrakhiani/FLAREr` | repo to install FLAREr from |
| `FLARER_VERSION` | `flare-io-on-netcdf-v2` | FLAREr branch / tag / commit |
| `GHCR_IO_REPO` | *(your namespace)* | GHCR namespace to push the image to |

To build it:

1. In the FaaSr-Docker fork, go to **Actions** and select
   **build-glm-aed-flare-rs-container -> GHCR**.
2. Click **Run workflow** and adjust the inputs if needed (for example, point
   `FLARER_INSTALL_REPO` / `FLARER_VERSION` at your own fork and branch).
3. Run it. The workflow logs in to GHCR, builds the image, and pushes it with both the
   version tag and `latest`, e.g.:
   ```text
   ghcr.io/<your-namespace>/github-actions-glm-aed-flare-rs:2.0.5
   ghcr.io/<your-namespace>/github-actions-glm-aed-flare-rs:latest
   ```

The workflow authenticates to GHCR with the repository's built-in `GITHUB_TOKEN` and also
passes that token to the build as `GITHUB_PAT`, so no extra secret is required for the
`install_github` steps.

## Using the image in a workflow

Once the image is published, reference it per-action in your workflow JSON under
`ActionContainers`, mapping each action name to your image:

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

The FLARE image demonstrates patterns you can apply to any custom scientific-computing
image:

- **Multi-stage compile-from-source** — build a binary in a throwaway stage and copy only
  the result into the runtime image, keeping the toolchain out of the published image.
- **A specialized base image** — start from a base such as `rocker/geospatial` that
  already carries heavy system and language dependencies, and install the FaaSr runtime
  into it.
- **Parameterized installs** — expose install repositories and versions as build
  arguments so one Dockerfile builds multiple variants (canonical vs. your fork).
- **A package manifest file** — keep long dependency lists out of the Dockerfile.
- **A token for `install_github`** — pass `GITHUB_PAT` to avoid the anonymous API rate
  limit during the build.
- **A custom entry point** — ship your own `faasr_entry.py` when the function needs
  bespoke startup behavior.

[Advanced usage]: advanced.md
[Building containers]: docker_build.md
[FaaSr Workflow Builder Web UI]: workflows.md
[Creating and editing workflows]: workflows.md
[register the workflow]: register_workflow.md
[security best practices]: security.md
