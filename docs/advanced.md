# Advanced usage

## Creating custom container images

FaaSr provides a set of default base container images (see [default values]) that already include the FaaSr runtime and a minimal set of dependences for both Python and R. These are sufficient for most workflows. Before building a custom image, consider whether your needs can be met by simply declaring [package dependences] in your workflow JSON - CRAN, PyPI, and GitHub packages listed there are downloaded and installed automatically before your function runs, without requiring you to build or maintain a container.

You may need a custom container image when:

- Your function depends on system libraries that must be installed with `apt` (these cannot be installed through the dynamic [package dependences] mechanism)
- You want heavy or slow-to-install packages baked into the image ahead of time, to reduce action startup time
- You need a fixed, reproducible environment with pinned package versions
- You depend on a base image other than the FaaSr defaults

### Where the images are built

All FaaSr container images are built from the [FaaSr-Docker repository](https://github.com/FaaSr/FaaSr-Docker). To build your own, fork this repository to your GitHub account and configure the registry secrets it needs. The full build process - including the GitHub Actions used to build and publish each image, and the registry credentials required - is described in the [Building containers] documentation.

FaaSr images come in two layers:

- A **base** image, which contains the language runtime (Python, or R from Rocker) plus a common set of dependences
- A **platform-specific** image, which builds _from_ a base image and adds the code needed to run on a particular provider (GitHub Actions, AWS Lambda, GCP, OpenWhisk, or Slurm)

To customize an image with your own dependences, you edit the base image dependences, rebuild the base image, and then rebuild the platform-specific image(s) from your customized base.

### Adding dependences to a base image

The dependences installed into a base image are declared in three files in the `base/` folder of the [FaaSr-Docker repository](https://github.com/FaaSr/FaaSr-Docker):

- `apt-packages.txt` - system packages installed with `apt` (one package per line)
- `requirements.txt` - Python packages installed with `pip` (one package per line, optionally pinned to a version)
- `R_packages.R` - R packages, installed with pinned versions via `remotes::install_version()`

Add the packages your functions require to the appropriate file(s), then rebuild the base image. When you run the build action, you can also change the upstream image your base is built from (the `BUILD_FROM` input, e.g. `python:3.13` or `rocker/tidyverse:4.4`) and give your base a descriptive name (the `BASE_NAME` input, e.g. `base-geospatial`).

!!! note
    Use the `rocker-base` build for images that run R functions, and the `py-base` build for Python-only images. Although a Rocker base can technically run both, it is recommended for R functions.

### Customizing platform-specific Dockerfiles

The platform-specific Dockerfiles live in the `faas_specific/` folder. If you need to add dependences at this layer, you may edit these files to add them - but you **must not** modify the `WORKDIR`, `ARG`, `RUN`, `COPY`, `FROM`, and `CMD` statements, as these are required for the container to run correctly under FaaSr. In most cases it is preferable to add your dependences to the base image instead.

### Going further: a fully custom image

The customization above keeps the standard FaaSr build structure. When a function needs a binary compiled from source, a package installed from a specific Git branch, or a custom entry point, you combine a customized base image with a dedicated Dockerfile. The [Building a custom image] page describes the recommended path for doing this, using the GLM-AED-FLARE lake-forecast image as a worked example.

### Using a custom image in a workflow

Once your custom image is built and published to a container registry (e.g. DockerHub, GHCR, or ECR), you can use it in a workflow:

- In the [FaaSr Workflow Builder Web UI], select the Action you want to run in your custom container, and specify your image under **Action Containers** (see [Creating and editing workflows])
- When you [register the workflow] with `FAASR REGISTER`, you are required to tick the `Allow custom containers` checkbox, since the action detects the use of a non-default container

!!! warning
    Only run custom containers that you build yourself or whose provenance you trust. Review the [security best practices] before using custom containers.

## Custom compute server names and register/invoke actions

[default values]: defaults.md
[package dependences]: dependences.md
[Building a custom image]: custom_image.md
[Building containers]: docker_build.md
[FaaSr Workflow Builder Web UI]: workflows.md
[Creating and editing workflows]: workflows.md
[register the workflow]: register_workflow.md
[security best practices]: security.md
