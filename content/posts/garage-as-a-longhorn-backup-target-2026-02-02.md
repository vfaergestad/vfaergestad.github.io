---
title: "Minio S3 as a Longhorn Backup Target"
date: 2026-02-02T21:54:30+02:00
draft: false
---

## Why

In https://blog.faergestad.com/posts/minio-as-a-longhorn-backup-target-2024-10-24/ I wrote about using minio as a longhorn backup target. Minio has since gone evil[0], so lets switch it out with garage.
Garage is a service that provides selfhost-able S3 storage, compatible with the AWS S3 API.

## What

- [Garage](https://git.deuxfleurs.fr/Deuxfleurs/garage)
- [Longhorn](https://github.com/longhorn/longhorn)
- [Kubernetes](https://kubernetes.io/) (I'm using [talos](https://github.com/siderolabs/talos))

## What it's not

- How to set up Longhorn
- How to set up garage
- How to set up Kubernetes.

## How

### Prerequisites

- Access to a Kubernetes cluster using `kubectl`.
- Longhorn installed on the Kubernetes-cluster.
- Garage running somewhere reachable by Longhorn.

### Access Garage

If you have garage installed, you probably know how to access it. I use the web ui.

### The Bucket

First, create a bucket in garage where the backups should reside. I called it longhorn-backup.  


### Authentication

Next, create a key that Longhorn can use to access the bucket, and give it the `readwrite` policy:

Please write the credentials down somewhere (1password).

Give the key access to the bucket.

### Longhorn-settings

Here comes the part that prompted me to make this post, since I got stuck here.

You need to create a secret in kubernetes with the following keys:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: garage-secret # Or something else
  namespace: longhorn-system # namespace of longhorn
type: Opaque
data:
  AWS_ACCESS_KEY_ID: Access key of the created user.
  AWS_SECRET_ACCESS_KEY: Secret key of the created user.
  AWS_ENDPOINTS: The endpoint of your garage-server, for example http://garage.example.org.
```

Remember that the values of these keys needs to be base64 encoded.

This key can be created with kubectl:

```shell
kubectl create secret generic garage-secret \
  --namespace=longhorn-system \
  --from-literal=AWS_ACCESS_KEY_ID=<your-access-key-id> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<your-secret-access-key> \
  --from-literal=AWS_ENDPOINTS=<your-endpoints>
```

Or, even better, be created from an external secret manager, like 1password (i should really write about this sometime).

The last step, set the backup-target in longhorn. This may be done in the UI->Settings, or through Helm-values:

```yaml
defaultSettings:
  backupTarget: "s3://longhorn-backup@garage/" # <nameofbucket>@<somedummyregion>
  backupTargetCredentialSecret: garage-secret # This needs to be the same as the name of the secret
```

And you should be set! Don't hesitate to reach out if you run into any problems.

## Sources and related links

<https://longhorn.io/docs/archives/1.3.1/snapshots-and-backups/backup-and-restore/set-backup-target/#enable-virtual-hosted-style-access-for-s3-compatible-backupstore>

<https://longhorn.io/docs/archives/1.3.1/advanced-resources/deploy/customizing-default-settings/#using-helm>

