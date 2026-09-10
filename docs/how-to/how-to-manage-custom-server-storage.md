# How to Configure Control Plane Storage Backends

This guide explains how to configure storage backends for the self-hosted Ejenix control plane server across local disk, cloud storage buckets, and containerized deployments.

---

## 1. Storage Backend Overview

The control plane stores two types of data:
1. **Metadata & State Records**: Application registries, channel pointers (`(appId, channel, env)` -> active bundle ID), and promotion history.
2. **Bundle Artifact Files**: Signed CBOR bundle binaries (`.bundle`) and computed delta patches.

---

## 2. Deployment Target Storage Architectures

### Target 1: Local Disk & Docker (`--target docker`)

For single-instance VM deployments or local Docker:

- **Storage Type**: Local filesystem directory.
- **Path**: Mounts host volume to `/var/lib/ejenix/storage`.
- **Docker Mount**:
  ```yaml
  volumes:
    - ejenix-data:/var/lib/ejenix/storage
  ```

### Target 2: Google Cloud Storage (`--target gcp`)

For Google Cloud Run serverless deployments:

- **Storage Type**: GCS Storage Bucket (`GoogleCloudStorageProvider`).
- **Deployment Command**:
  ```bash
  ./deploy.sh --target gcp --bucket my-ejenix-bundles
  ```
- **Durability**: High availability, multi-region replication, zero server disk persistence required.

### Target 3: Azure Blob Storage (`--target azure`)

For Azure Container Apps deployments:

- **Storage Type**: Azure Blob Storage (`AzureBlobStorageProvider`).
- **Deployment Command**:
  ```bash
  ./deploy.sh --target azure --resource-group ejenix-rg --storage-account ejenixstorage
  ```

### Target 4: AWS ECS/Fargate (`--target aws`)

- **Important Notice**: AWS App Runner does **not** support persistent volume mounts (`--ephemeral`). Bundles stored on App Runner are lost on container restart.
- **Production AWS Solution**: Deploy the Ejenix Docker container on **AWS ECS / Fargate** backed by an **Amazon EFS** (Elastic File System) persistent volume mount.

### Target 5: Kubernetes (`--target kubernetes`)

For Kubernetes cluster deployments:

- **Storage Type**: `PersistentVolumeClaim` (PVC) bound to a durable storage class (e.g. AWS EBS, GCP Persistent Disk, Azure Disk, Longhorn).
- **Manifest (`deploy/k8s/control-plane.yaml`)**:
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: ejenix-storage-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  ```

---

## 3. Data Retention & Backup Best Practices

1. **Database / Metadata Backup**: Periodically back up the metadata state file (`storage/metadata.json`) or snapshot cloud storage buckets.
2. **Bundle Artifact Retention**: Bundle IDs are immutable content digests. Never delete active or previous bundle artifacts from storage while devices are running in the field.
