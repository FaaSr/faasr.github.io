# Setting up a Backblaze B2 data store

FaaSr passes files between actions through an **S3-compatible object store**. The [tutorial] and many examples use the public [MinIO Play](https://play.min.io) sandbox, but Play is a shared demo service that is frequently unavailable. This guide walks you through setting up **[Backblaze B2](https://www.backblaze.com/cloud-storage)** — a reliable, S3-compatible store with a free tier (10 GB) — as an alternative you fully control.

You only need to do this once; afterward any workflow can point at your Backblaze bucket as its data store.

## 1. Create a Backblaze account and a bucket

1. Sign up for **B2 Cloud Storage** at [backblaze.com](https://www.backblaze.com/cloud-storage) using your email.
2. In the left menu under **B2 Cloud Storage**, click **Buckets → Create a Bucket**.
3. Give the bucket a **globally unique name** (e.g. `myuniquebucket`) — you'll need this name later.
4. Set **Files in Bucket** to **Private**. Leave encryption and Object Lock **disabled** (the defaults).
5. Click **Create a Bucket**.

## 2. Create an application key

1. In the left menu, click **Application Keys → Add a New Application Key**.
2. Give it a name (e.g. `faasr-key`).
3. Under **Allow access to Bucket(s)**, select the bucket you just created (or all buckets).
4. Set **Type of Access** to **Read and Write**.
5. Click **Create New Key**.

!!! warning "Copy the key now"
    Backblaze shows the **`keyID`** and the **`applicationKey`** **only once**, on this screen. Copy both before leaving the page — you cannot retrieve the `applicationKey` again (you'd have to create a new key).

## 3. Note your endpoint and region

On your bucket's detail page, Backblaze lists an **Endpoint** such as:

```
s3.us-east-005.backblazeb2.com
```

- Your **Region** is the segment between `s3.` and `.backblazeb2.com` — here, `us-east-005`.
- Your full endpoint URL (used below) is that value with `https://` in front: `https://s3.us-east-005.backblazeb2.com`.

## 4. Store the credentials as GitHub Secrets

In your [FaaSr-workflow repo], add two [GitHub Secrets](credentials.md) named after your data store. FaaSr derives the secret names from the data store's name in the workflow: `<DataStoreName>_AccessKey` and `<DataStoreName>_SecretKey`.

For a data store named **`S3`** (as in the tutorial):

| Secret | Value |
| --- | --- |
| `S3_AccessKey` | your Backblaze **`keyID`** |
| `S3_SecretKey` | your Backblaze **`applicationKey`** |

(If you name your data store differently, e.g. `My_S3_Bucket`, the secrets are `My_S3_Bucket_AccessKey` / `My_S3_Bucket_SecretKey`.)

## 5. Configure the data store in your workflow

In the [workflow builder] use **Edit Data Store**, or edit the `DataStores` block of your workflow JSON directly:

```json
"DataStores": {
  "S3": {
    "Endpoint": "https://s3.us-east-005.backblazeb2.com",
    "Bucket": "myuniquebucket",
    "Region": "us-east-005",
    "Writable": "TRUE"
  }
},
"DefaultDataStore": "S3"
```

- **`Endpoint`** — `https://` + your Backblaze endpoint from step 3.
- **`Bucket`** — your bucket name from step 1.
- **`Region`** — the region from step 3 (e.g. `us-east-005`).
- **`Writable`** — `TRUE` so actions can write results back.

## 6. Use it in the tutorial

To run the [tutorial] against Backblaze instead of MinIO Play, simply replace the tutorial's data store with the block above (matching the data store **name** `S3` so the secret names line up), then register and invoke as usual. Your workflow's inputs and outputs will now live in your Backblaze bucket, which you can browse anytime from the Backblaze web console.

!!! tip "Free tier is plenty for the tutorial"
    Backblaze B2's free tier includes 10 GB of storage and a generous daily download allowance — far more than the tutorial or typical FaaSr examples need.

[tutorial]: tutorial.md
[workflow builder]: workflows.md
[FaaSr-workflow repo]: workflow_repo.md
