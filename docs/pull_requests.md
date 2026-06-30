# Process for issuing Pull Requests (PRs)

!!! note
    This page describes the general process for contributing code to FaaSr. The specific review and testing requirements for individual repositories are still evolving; if you are planning a substantial contribution, please [contact the team](contact.md) first so we can align on scope before you begin.

FaaSr development happens on GitHub, and we follow a fork-and-pull-request model. Pull requests are reviewed by one or more experienced peer developers before being merged, so we expect contributions to be of good quality and thoroughly tested.

## Before you start

- For changes that may require significant modifications to software modules, [contact the team](contact.md) before starting so we can understand the scope of the work.
- Identify which repository your change belongs to (see [Code repositories]) - for example the FaaSr-Backend engine, the [workflow builder][Developing the workflow builder], the container images in FaaSr-Docker, or this documentation site.

## Fork and branch

- Fork the relevant repository to your own GitHub account and work on a branch in your fork.
- For the FaaSr-workflow-builder, uncheck *Copy the main branch only* when forking so that the `gh-pages` branch is included (see [Developing the workflow builder]).

## Test before opening a pull request

- Test your changes thoroughly before opening a PR. We expect code to be well tested before a pull request is accepted.
- The FaaSr-Backend provides integration tests (under `FaaSr_py/testing/`, e.g. `workflow_test.py`) that you can use to validate your changes against a workflow.
- For the FaaSr-workflow-builder, set the `debug` constant in `WorkflowContext.js` back to `false` before submitting (see [Developing the workflow builder]).

## Open the pull request

- Open a pull request from your fork's branch against the upstream repository's default branch.
- For the FaaSr-workflow-builder, a merged pull request triggers a GitHub Action that builds the `src` branch into the `gh-pages` branch and deploys it to [faasr.io/FaaSr-workflow-builder](https://faasr.io/FaaSr-workflow-builder/).
- One or more experienced peer developers will review your PR. Address review feedback until the PR is approved and merged.

[Code repositories]: repositories.md
[Developing the workflow builder]: workflow_builder.md
