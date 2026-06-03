# Provisioning EphFlow VMs

## Overview

Serverless FaaS platforms impose strict resource limits - for example, AWS Lambda actions are capped at a 15-minute runtime and a fixed amount of memory. Some workflow actions (e.g. long-running scientific simulations) exceed these limits. _EphFlow_ is FaaSr's ephemeral VM orchestration capability: it lets a workflow provision a virtual machine on-demand to run resource-intensive actions, and tears the VM down when the workflow finishes - without requiring you to manually manage any persistent infrastructure, and without changing your function code.

The orchestration technique is **platform-agnostic by design**: EphFlow injects VM lifecycle actions (start, poll, stop) directly into the workflow DAG and manages instances through a provider abstraction, so it generalizes to any IaaS provider that exposes APIs for VM lifecycle management, paired with any execution backend a FaaSr [compute server] can target. The reference implementation documented here demonstrates the technique end-to-end using an [AWS EC2] instance that is pre-configured as a [GitHub Actions self-hosted runner]: actions you flag as VM-requiring run on the EC2 instance, while the rest of your workflow continues to run on serverless GitHub Actions runners. AWS EC2 with GitHub Actions self-hosted runners is the combination currently supported out of the box.

## How it works

EphFlow adds three pieces to a workflow:

- A top-level **`VMConfig`** in the workflow JSON that describes the EC2 instance to use
- A per-action **`RequiresVM`** flag that marks which actions must run on the VM
- A set of injected **VM orchestration actions** that manage the VM lifecycle

You do not write the orchestration actions yourself. Instead, a _VM injection tool_ reads your workflow, detects the actions flagged with `RequiresVM`, and produces an _augmented_ workflow JSON with three built-in actions inserted into the DAG:

- `vm_start` - injected at the workflow entry point; issues a non-blocking command to start the EC2 instance
- `vm_poll` - injected immediately before each VM-requiring action; waits until the instance is healthy and the self-hosted runner reports online
- `vm_stop` - injected after all leaf actions; stops the instance so it does not incur charges once the workflow completes

When you [register] the augmented workflow, each action flagged with `RequiresVM` is deployed as a GitHub Action that runs on the self-hosted runner (`runs-on: self-hosted`), while all other actions run on GitHub-hosted runners (`runs-on: ubuntu-latest`). At runtime, non-VM actions can proceed while the VM starts up in the background; the `vm_poll` action holds each VM-requiring action until the runner is ready.

## Prerequisites

Before configuring a workflow for VM orchestration, you need:

- An existing AWS EC2 instance, pre-configured to register itself as a [GitHub Actions self-hosted runner] attached to your [FaaSr-workflow repo]. The instance can be left in the stopped state - FaaSr starts and stops it as part of the workflow.
- AWS credentials (an access key and secret key) with permission to start, stop, and describe that EC2 instance.
- A GitHub Personal Access Token (`GH_PAT`), which `vm_poll` uses to verify that the self-hosted runner is online via the GitHub API.

## Configuring VMConfig

Add a top-level `VMConfig` object to your workflow JSON describing the instance:

```json
"VMConfig": {
    "Name": "AWS_VM",
    "Provider": "AWS",
    "InstanceId": "i-09405009da063a647",
    "Region": "us-west-2",
    "RunnerName": "ip-172-31-39-220"
}
```

- `Name` - a name for this VM configuration. It is also used to derive the names of the secrets that hold your AWS credentials (see [Secrets](#secrets) below). Required.
- `Provider` - the cloud provider for the VM. Currently only `AWS` is supported. Required.
- `InstanceId` - the ID of the existing EC2 instance to start and stop. Required.
- `Region` - the AWS region the instance is in. Required.
- `RunnerName` - the name of the self-hosted runner the instance registers as. Optional, but recommended: when provided (together with `GH_PAT`), `vm_poll` verifies through the GitHub API that the runner is online before allowing the VM-requiring action to run.

## Marking actions that require the VM

For each action that must run on the VM, set `RequiresVM` to `true` in the `ActionList`:

```json
"compute_intensive": {
    "FunctionName": "run_simulation",
    "FaaSServer": "GH",
    "Type": "R",
    "RequiresVM": true,
    "InvokeNext": []
}
```

Actions without this flag (or with `RequiresVM` set to `false`) run on serverless GitHub-hosted runners as usual. Only actions on a `GitHubActions` compute server may require a VM.

## Secrets

Store the AWS credentials for the VM as GitHub Actions secrets in your [FaaSr-workflow repo], named after the `VMConfig.Name`. For a `VMConfig.Name` of `AWS_VM`, the two secrets are:

- `AWS_VM_AccessKey` - the AWS access key
- `AWS_VM_SecretKey` - the AWS secret key

You also need your `GH_PAT` secret (see [creating cloud credentials]) for runner verification. These VM credentials are independent of the `AWS_AccessKey`/`AWS_SecretKey` you would use for AWS Lambda compute servers.

## Augmenting the workflow

Once your workflow JSON declares `VMConfig` and one or more `RequiresVM` actions, run the VM injection tool to produce the augmented workflow. The tool supports two strategies:

- **`parallel`** (default) - `vm_start` is non-blocking, and a `vm_poll` action is inserted before each VM-requiring action. Non-VM actions execute concurrently while the VM starts, and each VM action waits only at its own poll. This is the more efficient strategy.
- **`sequential`** - `vm_start` blocks at the entry point until the VM is fully ready, and the rest of the workflow runs afterwards. This is the simpler strategy.

The tool reads `your-workflow.json` and writes `your-workflow_augmented.json`, leaving your original file unchanged. If the workflow has no `RequiresVM` actions, the tool passes it through without modification.

!!! note
    Always register and invoke the **augmented** JSON (e.g. `your-workflow_augmented.json`), not the original. The augmented file is the one that contains the `vm_start`, `vm_poll`, and `vm_stop` actions.

## Registering and invoking

After producing the augmented workflow, proceed as with any FaaSr workflow:

- [Register] the augmented JSON. Actions flagged with `RequiresVM` are created as self-hosted runner actions; the injected `vm_start`, `vm_poll`, and `vm_stop` actions are created as well.
- [Invoke] the augmented JSON to run the workflow. The entry point is `vm_start`, which starts the EC2 instance; the workflow then proceeds according to the augmented DAG, and `vm_stop` shuts the instance down after the final actions complete.

## Notes and constraints

- In the current reference implementation, VM-requiring actions run on a `GitHubActions` compute server: the VM is provisioned as a GitHub Actions self-hosted runner, so the `RequiresVM` flag has no effect on actions assigned to Lambda, GCP, OpenWhisk, or Slurm. The underlying technique is not specific to GitHub Actions (see [Overview](#overview)) - this is the combination supported today.
- The provider abstraction currently ships with an AWS implementation; `VMConfig.Provider` accepts `AWS`.
- VM injection preserves the structure of your DAG, including [conditional invocations]; the augmented workflow is still a valid [DAG].
- `vm_stop` is injected after all leaf actions and is designed to run regardless of whether upstream actions succeeded, so the instance is not left running.
- `vm_poll` waits up to five minutes for the instance to pass its status checks, and up to a further period for the self-hosted runner to report online through the GitHub API.

[AWS EC2]: https://aws.amazon.com/ec2/
[GitHub Actions self-hosted runner]: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners
[compute server]: workflows.md
[register]: register_workflow.md
[Register]: register_workflow.md
[Invoke]: invoke_workflow.md
[FaaSr-workflow repo]: workflow_repo.md
[creating cloud credentials]: credentials.md
[conditional invocations]: conditional.md
[DAG]: prog_model.md
