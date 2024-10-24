---
title: "Minio S3 as a Longhorn Backup Target"
date: 2024-10-24T19:40:30+02:00
draft: false
---

## Why

Longhorn is a great alternative for achieving cloud-like storage in Kubernetes, and you would want to a reliable way of backing up the data you store there.
Minio is a service that provides selfhost-able S3 storage, compatible with the AWS S3 API.

## What

- [Minio](https://github.com/minio/minio)
- [Longhorn](https://github.com/longhorn/longhorn)
- [Kubernetes](https://kubernetes.io/) (I'm using [k3s](https://k3s.io/))

## What it's not

- How to set up Longhorn
- How to set up Minio
- How to set up Kubernetes.

## How

### Prerequisites

- Access to a Kubernetes cluster using `kubectl`.
- Longhorn installed on the Kubernetes-cluster.
- Minio running somewhere reachable by Longhorn.

### Access Minio

If you have minio installed, you probably know how to access it, but hey, you may just have installed it for this post, so:

You can access minio through either the GUI or the CLI. For the CLI, it can be handy to create an alias first: (I use `mcli`, but your command may be `mc`)

```shell
mcli alias set myminio/ http://MINIO-SERVER MYUSER MYPASSWORD
```

For the GUI, well, go to the URL and login.
The rest of the instructions will be for the CLI, but you will find your way around in the GUI yourself. It is intuitive.

### The Bucket

First, create a bucket in Minio where the backups should reside.

```shell
mcli mb myminio/longhorn-backup
```

You can use the `ls` command to list your buckets:

```shell
mcli ls myminio
[2024-10-23 21:33:50 CEST]     0B longhorn-backup/
```

### Authentication

Next, create a user that Longhorn can use to access the bucket, and give it the `readwrite` policy:

```shell
mcli admin user add myminio ACCESSKEY SECRETKEY

mcli admin policy attach myminio readwrite --user=ACCESSKEY
```

Please write the credentials down somewhere (1password).

You can list the users to give yourself a pat on the back:

```shell
mcli admin user list myminio

```

The user will by default have access to the created bucket.

### Longhorn-settings

Here comes the part that prompted me to make this post, since I got stuck here.

You need to create a secret in kubernetes with the following keys:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-secret # Or something else
  namespace: longhorn-system # namespace of longhorn
type: Opaque
data:
  AWS_ACCESS_KEY_ID: Access key of the created user.
  AWS_SECRET_ACCESS_KEY: Secret key of the created user.
  AWS_ENDPOINTS: The endpoint of your minio-server, for example http://minio.example.org.
```

Remember that the values of these keys needs to be base64 encoded.

This key can be created with kubectl:

```shell
kubectl create secret generic minio-secret \
  --namespace=longhorn-system \
  --from-literal=AWS_ACCESS_KEY_ID=<your-access-key-id> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<your-secret-access-key> \
  --from-literal=AWS_ENDPOINTS=<your-endpoints>
```

Or, even better, be created from an external secret manager, like 1password (i should really write about this sometime).

The last step, set the backup-target in longhorn. This may be done in the UI->Settings, or through Helm-values:

```yaml
defaultSettings:
  backupTarget: "s3://longhorn-backup@us-east-1/" # <nameofbucket>@<somedummyregion>
  backupTargetCredentialSecret: minio-secret # This needs to be the same as the name of the secret
```

And you should be set! Don't hesitate to reach out if you run into any problems.

## Sources and related links

<https://min.io/docs/minio/linux/administration/identity-access-management/minio-user-management.html>

<https://longhorn.io/docs/archives/1.3.1/snapshots-and-backups/backup-and-restore/set-backup-target/#enable-virtual-hosted-style-access-for-s3-compatible-backupstore>

<https://longhorn.io/docs/archives/1.3.1/advanced-resources/deploy/customizing-default-settings/#using-helm>

