# Running FaaSr on Kubernetes (NRP)

This guide covers what you need to run FaaSr workflows on a **Kubernetes** cluster, including the [National Research Platform (NRP)](https://nationalresearchplatform.org/). It assumes you have already set up your [FaaSr-workflow repo] and are familiar with the [tutorial]. For Kubernetes/NRP account and cluster access itself, refer to your cluster's or NRP's own documentation.

## Overview

When you register a workflow with a Kubernetes compute server, `FAASR REGISTER` creates one **Kubernetes Job per action** (named `WorkflowName-ActionName`) by talking directly to the cluster's API server with a bearer token. Invoking the workflow runs those Jobs in your namespace, reading and writing their data to your S3 data store. NRP (Nautilus) is a Kubernetes cluster, so running on NRP is the same as running on any Kubernetes cluster — you point FaaSr at the NRP API server and your NRP namespace.

## Prerequisites

- Access to a **Kubernetes cluster** (or an **NRP/Nautilus** account).
- A **namespace** on that cluster where your Jobs will run (e.g. `faasr-jobs`).
- The cluster's **API server endpoint** (e.g. `https://<host>:6443`).
- A **service-account token** (a JWT) with permission to create and run Jobs in that namespace.

## Credentials and secrets

FaaSr authenticates to the cluster API with a JWT service-account token, stored as a GitHub Secret in your [FaaSr-workflow repo] (see [Creating cloud credentials]):

| Secret | Value |
| --- | --- |
| `<ServerName>_Token` | the JWT service-account token for your namespace |

The secret name is your compute server's name followed by `_Token` — for example, if you name the compute server `K8s`, the secret is `K8s_Token`.

## Container image

Kubernetes actions run a FaaSr Kubernetes image, for example `faasr/kubernetes-python` (or `faasr/kubernetes-r`); you can also use your own image. Reference it under **Action Containers** in the [workflow builder] (or `ActionContainers` in the workflow JSON). The image must be pullable by your cluster.

## Configuring the compute server

In the [workflow builder], use **Edit Compute Servers** to add a Kubernetes server. The configuration fields are:

| Field | Meaning | Example |
| --- | --- | --- |
| `FaaSType` | must be `Kubernetes` | `Kubernetes` |
| `Endpoint` | the cluster's API server URL | `https://129.146.139.205:6443` |
| `Namespace` | the namespace your Jobs run in | `faasr-jobs` |
| `AllowSelfSignedCertificate` | allow a self-signed cluster certificate (optional) | `true` |
| `SSLCertificate` | the cluster's SSL/CA certificate, if required (optional) | *(certificate)* |
| `CPUsPerTask` | max CPU per container, in **millicores** (optional) | `1000` |
| `Memory` | memory per container, in MB (optional) | `512` |

Minimal JSON (token supplied via the secret store):

```json
"ComputeServers": {
  "K8s": {
    "FaaSType": "Kubernetes",
    "Endpoint": "https://129.146.139.205:6443",
    "Namespace": "faasr-jobs",
    "UseSecretStore": true
  }
}
```

## Register and invoke

Once the secret, container image, and compute server are set:

1. Upload your workflow JSON to your [FaaSr-workflow repo].
2. Run **`FAASR REGISTER`** — this creates a Kubernetes Job per action (`WorkflowName-ActionName`) in your namespace (see [Registering workflows]).
3. Run **`FAASR INVOKE`** to execute the workflow (see [Invoking workflows]).

## Running on NRP (Nautilus)

NRP is a Kubernetes cluster, so the steps above apply directly:

- Obtain a **namespace** and a **service-account token** from NRP for your project, and the NRP **API server endpoint**.
- Store the token as `<ServerName>_Token`, and set `Endpoint` and `Namespace` accordingly.
- If NRP's API server presents a self-signed or custom certificate, set `AllowSelfSignedCertificate` (or provide the `SSLCertificate`) as needed.

## Notes

- **Token expiry:** service-account JWTs are often time-limited. If registration or invocation fails with an authentication error, your token may have expired — obtain a fresh one and update the `<ServerName>_Token` secret.
- **CPU units:** `CPUsPerTask` is specified in **millicores** (e.g. `1000` = 1 vCPU), following Kubernetes conventions.
- **Namespace permissions:** the service account behind the token needs permission to create, run, and delete Jobs in the target namespace.

[FaaSr-workflow repo]: workflow_repo.md
[tutorial]: tutorial.md
[Creating cloud credentials]: credentials.md
[workflow builder]: workflows.md
[Registering workflows]: register_workflow.md
[Invoking workflows]: invoke_workflow.md
