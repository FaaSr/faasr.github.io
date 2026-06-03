# Code repositories

FaaSr is open-source software released under the MIT license. All of the source code is hosted on GitHub under the [FaaSr organization](https://github.com/FaaSr). This page is a directory of the project's repositories, so that developers and reviewers can find the implementation behind each part of the system.

## Core components

- **[FaaSr-Backend](https://github.com/FaaSr/FaaSr-Backend)** - the core middleware that runs inside every action container (the `FaaSr_py` package). It contains the container entry point, the local RPC server and language client bindings, the cross-platform scheduler/dispatchers (GitHub Actions, AWS Lambda, Google Cloud, OpenWhisk, Slurm), the S3 data and logging APIs, the workflow JSON schema, and the ephemeral VM orchestration code (the `vm/` provider abstraction and the `vm_start` / `vm_poll` / `vm_stop` built-in functions).
- **[FaaSr-Docker](https://github.com/FaaSr/FaaSr-Docker)** - the Dockerfiles and GitHub Actions used to build and publish the base and platform-specific container images for each supported provider. See [Building containers](docker_build.md).
- **[FaaSr-workflow](https://github.com/FaaSr/FaaSr-workflow)** - the control-plane repository that users fork to manage their workflows. It holds the `FAASR REGISTER`, `FAASR INVOKE`, and secret-sync GitHub Actions and their scripts, as well as the VM injection tool that augments workflows with VM orchestration actions. See [Setting up the FaaSr-workflow repo](workflow_repo.md).
- **[FaaSr-workflow-builder](https://github.com/FaaSr/FaaSr-workflow-builder)** - the React web application that composes and edits FaaSr-compliant workflow JSON files, deployed at [faasr.io/FaaSr-workflow-builder](https://faasr.io/FaaSr-workflow-builder/). See [Creating and editing workflows](workflows.md).

## Examples and documentation

- **[FaaSr-Functions](https://github.com/FaaSr/FaaSr-Functions)** - a collection of example workflows (JSON) and R/Python functions that serve as templates, including the workflows used in the [tutorial](tutorial.md). See [Example workflows and functions](functionexamples.md).
- **[faasr.github.io](https://github.com/FaaSr/faasr.github.io)** - the source for this documentation site, built with MkDocs and the Material theme.
