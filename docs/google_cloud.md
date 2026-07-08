# Running FaaSr on Google Cloud

This guide covers what you need to run FaaSr workflows on **Google Cloud**, where each action runs as a **Cloud Run job**. It assumes you have already set up your [FaaSr-workflow repo] and are familiar with the [tutorial]. For Google Cloud account setup itself, refer to the [Google Cloud documentation](https://cloud.google.com/run/docs).

## Overview

When you register a workflow with a Google Cloud compute server, `FAASR REGISTER` creates one **Cloud Run job per action** (named `WorkflowName-ActionName`), using a container image from a container registry. Invoking the workflow runs those jobs, which read and write their data to your S3 data store.

## Prerequisites

- A **Google Cloud project** with the **Cloud Run API** enabled.
- A **service account** in that project with permission to create and run Cloud Run jobs.
- A **service account key** (JSON) for that service account.
- A container image available in a registry Cloud Run can pull from (see [Container image](#container-image)).

## Credentials and secrets

Store the service account key as a GitHub Secret in your [FaaSr-workflow repo] (see [Creating cloud credentials]):

| Secret | Value |
| --- | --- |
| `GCP_SecretKey` | the service account key for your project |

The service account's email and token endpoint are supplied through the compute-server configuration (below), not as secrets.

## Container image

FaaSr provides `gcp-python` / `gcp-r` images (published to DockerHub as, e.g., `faasr/gcp-python:latest`); you can also build and publish your own with the `gcp -> DockerHub` action described in [Building containers]. Reference the image under **Action Containers** in the [workflow builder] (or `ActionContainers` in the workflow JSON).

## Configuring the compute server

In the [workflow builder], use **Edit Compute Servers** to add a Google Cloud server. The default compute server name for Google Cloud is `GCP`. The configuration fields are:

| Field | Meaning | Example |
| --- | --- | --- |
| `FaaSType` | must be `GoogleCloud` | `GoogleCloud` |
| `Namespace` | your GCP **project ID** | `my-gcp-project-123456` |
| `Region` | region for the Cloud Run jobs | `us-central1` |
| `Endpoint` | Cloud Run API endpoint | `https://run.googleapis.com/v2/projects/` |
| `ClientEmail` | the service account's email | `faasr@my-project.iam.gserviceaccount.com` |
| `TokenUri` | OAuth2 token endpoint | `https://oauth2.googleapis.com/token` |
| `Memory` | memory per job, in MB | `512` |
| `CPUsPerTask` | vCPUs allocated per job | `1` |
| `TimeLimit` | per-invocation timeout, in seconds | `3600` |

Equivalent JSON:

```json
"ComputeServers": {
  "GCP": {
    "FaaSType": "GoogleCloud",
    "Namespace": "my-gcp-project-123456",
    "Region": "us-central1",
    "Endpoint": "https://run.googleapis.com/v2/projects/",
    "UseSecretStore": true,
    "ClientEmail": "faasr@my-project.iam.gserviceaccount.com",
    "TokenUri": "https://oauth2.googleapis.com/token",
    "Memory": 512,
    "CPUsPerTask": 1,
    "TimeLimit": 3600
  }
}
```

## Register and invoke

Once the secret, container image, and compute server are set:

1. Upload your workflow JSON to your [FaaSr-workflow repo].
2. Run **`FAASR REGISTER`** — this creates a Cloud Run job per action (`WorkflowName-ActionName`) from your container image (see [Registering workflows]).
3. Run **`FAASR INVOKE`** to execute the workflow (see [Invoking workflows]).

## Notes

- **`ClientEmail` and `GCP_SecretKey` must match** — the email in the compute-server config must be the service account whose key you stored as `GCP_SecretKey`.
- **Permissions:** the service account needs to be able to create and run Cloud Run jobs in the project; if your S3 data store is on Google Cloud Storage, it also needs access there.
- **Timeout:** Cloud Run jobs allow longer runtimes than AWS Lambda (the `TimeLimit` above is in seconds), which makes GCP a good choice for longer-running actions.
- **Provider action logs:** in addition to FaaSr's `faasr_log` output in S3, per-job logs are available in Google Cloud Logging (see [Retrieving logs]).

[FaaSr-workflow repo]: workflow_repo.md
[tutorial]: tutorial.md
[Creating cloud credentials]: credentials.md
[Building containers]: docker_build.md
[workflow builder]: workflows.md
[Registering workflows]: register_workflow.md
[Invoking workflows]: invoke_workflow.md
[Retrieving logs]: logs.md
