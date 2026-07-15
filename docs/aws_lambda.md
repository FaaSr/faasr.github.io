# Running FaaSr on AWS Lambda

This guide covers what you need to run FaaSr workflows on **AWS Lambda**: the credentials to set up, the container image, how to configure the compute server, and how to register and invoke. It assumes you have already set up your [FaaSr-workflow repo] and are familiar with the [tutorial]. For AWS account setup itself, refer to the [AWS documentation](https://docs.aws.amazon.com/lambda/).

## Overview

When you register a workflow with an AWS Lambda compute server, `FAASR REGISTER` creates one **AWS Lambda function per action** (named `WorkflowName-ActionName`), using a container image stored in **Amazon ECR**. Invoking the workflow triggers those Lambda functions, which read and write their data to your S3 data store.

## Prerequisites

- An **AWS account**.
- An **IAM user** with programmatic access (an access key and secret key) and permissions to manage Lambda and ECR.
- A **Lambda execution role** — an IAM role that your Lambda functions assume at runtime. You will provide its ARN to FaaSr.
- A container image available in **Amazon ECR** in the region you will use (see [Container image](#container-image)).

## Credentials and secrets

Create the following as GitHub Secrets in your [FaaSr-workflow repo] (see [Creating cloud credentials]):

| Secret | Value |
| --- | --- |
| `AWS_AccessKey` | your IAM user's access key ID |
| `AWS_SecretKey` | your IAM user's secret access key |
| `AWS_ARN` | the ARN of the Lambda execution role your functions will assume |

## Container image

AWS Lambda pulls its container image from **Amazon ECR** (Lambda does not use DockerHub/GHCR for the runtime image). You need a FaaSr Lambda image in an ECR repository in the **same region** as your functions, for example:

```
<your-account-id>.dkr.ecr.us-east-1.amazonaws.com/aws-lambda-python:latest
<your-account-id>.dkr.ecr.us-east-1.amazonaws.com/aws-lambda-r:latest
```

FaaSr provides the `aws-lambda-python` / `aws-lambda-r` images; you can build and publish them to your own ECR registry using the `aws-lambda -> ECR` action described in [Building containers]. Reference the full ECR image URI under **Action Containers** in the [workflow builder] (or `ActionContainers` in the workflow JSON).

## Configuring the compute server

In the [workflow builder], use **Edit Compute Servers** to add an AWS Lambda server. The default compute server name for AWS Lambda is `AWS`. The configuration fields are:

| Field | Meaning | Example |
| --- | --- | --- |
| `FaaSType` | must be `Lambda` | `Lambda` |
| `Region` | AWS region for your functions and ECR image | `us-east-1` |
| `Memory` | memory per function, in MB | `512` |
| `CPUsPerTask` | vCPUs allocated per function | `1` |
| `TimeLimit` | per-invocation timeout, in seconds (Lambda's maximum is **900**) | `900` |

Equivalent JSON:

```json
"ComputeServers": {
  "AWS": {
    "FaaSType": "Lambda",
    "Region": "us-east-1",
    "UseSecretStore": true,
    "Memory": 512,
    "CPUsPerTask": 1,
    "TimeLimit": 900
  }
}
```

## Register and invoke

Once the secrets, container image, and compute server are set:

1. Upload your workflow JSON to your [FaaSr-workflow repo].
2. Run **`FAASR REGISTER`** — this creates a Lambda function per action (`WorkflowName-ActionName`) from your ECR image (see [Registering workflows]).
3. Run **`FAASR INVOKE`** to execute the workflow (see [Invoking workflows]).

## Notes and limits

- **Timeout:** Lambda functions are capped at **15 minutes** (`TimeLimit` ≤ 900). For longer-running work, use a different compute server (e.g. GitHub Actions, GCP, or Kubernetes).
- **Region consistency:** the ECR image, the Lambda functions, and the `Region` field must all be in the same region.
- **Execution role:** the role behind `AWS_ARN` needs the permissions your functions require (at minimum, basic Lambda execution and any access needed for your S3 data store if it is on AWS).
- **Provider action logs:** in addition to FaaSr's `faasr_log` output in S3, per-function logs are available in AWS CloudWatch (see [Retrieving logs]).

[FaaSr-workflow repo]: workflow_repo.md
[tutorial]: tutorial.md
[Creating cloud credentials]: credentials.md
[Building containers]: docker_build.md
[workflow builder]: workflows.md
[Registering workflows]: register_workflow.md
[Invoking workflows]: invoke_workflow.md
[Retrieving logs]: logs.md
