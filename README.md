# KubeBlocks Storage Providers

This repository contains a collection of `StorageProvider` Custom Resources (CRs) for [KubeBlocks](https://github.com/apecloud/kubeblocks), enabling backup and restore functionality with various storage backends.

## What is StorageProvider?

`StorageProvider` is a Custom Resource Definition (CRD) in KubeBlocks that represents storage providers for backup repositories. It defines how different storage systems can be accessed and configured for storing database backups.

A `StorageProvider` encapsulates:
- **Storage access methods**: CSI driver configurations, [datasafed](https://github.com/apecloud/datasafed) configurations
- **Access parameters**: Credentials, endpoints, buckets, and other provider-specific settings
- **Parameter schemas**: Validation rules and descriptions for required/optional parameters

For detailed API reference, see the [official documentation](https://kubeblocks.io/docs/release-0_9/user_docs/developer/api-reference/backup#dataprotection.kubeblocks.io/v1alpha1.StorageProvider).

## Supported Storage Providers

This repository includes `StorageProvider` definitions for the following storage backends:

| Provider | Description | File |
|----------|-------------|------|
| **S3** | Amazon S3 | [s3.yaml](storageprovider/s3.yaml) |
| **S3-Compatible** | S3-compatible storage | [s3-compatible.yaml](storageprovider/s3-compatible.yaml) |
| **MinIO** | MinIO object storage | [minio.yaml](storageprovider/minio.yaml) |
| **OSS** | Alibaba Cloud OSS | [oss.yaml](storageprovider/oss.yaml) |
| **COS** | Tencent Cloud COS | [cos.yaml](storageprovider/cos.yaml) |
| **OBS** | Huawei Cloud OBS | [obs.yaml](storageprovider/obs.yaml) |
| **GCS** | Google Cloud Storage (S3-compatible) | [gcs-s3comp.yaml](storageprovider/gcs-s3comp.yaml) |
| **Azure Blob** | Azure Blob Storage | [azureblob.yaml](storageprovider/azureblob.yaml) |
| **NFS** | Network File System | [nfs.yaml](storageprovider/nfs.yaml) |
| **FTP** | FTP/SFTP server | [ftp.yaml](storageprovider/ftp.yaml) |
| **PVC** | Kubernetes Persistent Volume Claim | [pvc.yaml](storageprovider/pvc.yaml) |

## How to Use

### Prerequisites

- KubeBlocks installed in your Kubernetes cluster
- (Optional) CSI driver - Only required if you choose the **Mount** access method:
  - [yandex-s3-csi-driver](https://github.com/yandex-cloud/k8s-csi-s3) - For S3-compatible providers (S3, S3-Compatible, OSS, COS, OBS, GCS, MinIO)
  - [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs) - For NFS provider

### Access Methods

KubeBlocks supports two methods to access remote object storage:

| Method | Description | CSI Driver Required | Security Consideration |
|--------|-------------|---------------------|------------------------|
| **Tool** | Uses command-line tool `datasafed` to directly access remote storage | ❌ No | Synchronizes credentials as secrets across namespaces |
| **Mount** | Mounts remote storage locally using a CSI driver | ✅ Yes | No credential sharing between namespaces |

**Recommendation**:
- Use **Tool** method for simpler setups without CSI driver installation
- Use **Mount** method for enhanced security in multi-tenant scenarios

The access method is specified in the `accessMethod` field when creating a BackupRepo. For more details, refer to the [BackupRepo documentation](https://kubeblocks.io/docs/release-0_9/user_docs/maintenance/backup-and-restore/backup/backup-repo).

### Installation

1. **Apply StorageProvider CRs**

   Install one or more storage providers based on your needs:

   ```bash
   # Install a specific provider (e.g., MinIO)
   kubectl apply -f storageprovider/minio.yaml

   # Or install all providers
   kubectl apply -f storageprovider/
   ```

2. **Verify Installation**

   ```bash
   kubectl get storageproviders.dataprotection.kubeblocks.io
   ```

   Example output:
   ```
   NAME            STATUS     CSIDRIVER          AGE
   azureblob       Ready                         30s
   cos             Ready      ru.yandex.s3.csi   30s
   ftp             Ready                         30s
   gcs-s3comp      Ready      ru.yandex.s3.csi   30s
   minio           Ready      ru.yandex.s3.csi   30s
   nfs             NotReady   nfs.csi.k8s.io     30s
   obs             Ready      ru.yandex.s3.csi   30s
   oss             Ready      ru.yandex.s3.csi   30s
   pvc             Ready                         30s
   s3              Ready      ru.yandex.s3.csi   30s
   s3-compatible   Ready      ru.yandex.s3.csi   30s
   ```

### Creating a BackupRepo

After installing a `StorageProvider`, you can create a `BackupRepo` to use it for backups. A `BackupRepo` references a `StorageProvider` and provides the actual configuration values.

**Example: Creating a BackupRepo with MinIO**

```yaml
# Create a secret to save the access key for MinIO
kubectl create secret generic minio-credential-for-backuprepo \
  -n kb-system \
  --from-literal=accessKeyId=<ACCESS KEY> \
  --from-literal=secretAccessKey=<SECRET KEY>

# Create the BackupRepo resource
kubectl apply -f - <<-'EOF'
apiVersion: dataprotection.kubeblocks.io/v1alpha1
kind: BackupRepo
metadata:
  name: my-repo
  annotations:
    dataprotection.kubeblocks.io/is-default-repo: "true"
spec:
  storageProviderRef: minio
  accessMethod: Tool
  pvReclaimPolicy: Retain
  volumeCapacity: 100Gi
  config:
    bucket: test-kb-backup
    mountOptions: ""
    endpoint: <ip:port>
  credential:
    name: minio-credential-for-backuprepo
    namespace: kb-system
  pathPrefix: ""
EOF

```

**Example: Creating a BackupRepo with AWS S3**

```yaml
# Create a secret to save the access key for S3
kubectl create secret generic s3-credential-for-backuprepo \
  -n kb-system \
  --from-literal=accessKeyId=<ACCESS KEY> \
  --from-literal=secretAccessKey=<SECRET KEY>

# Create the BackupRepo resource
kubectl apply -f - <<-'EOF'
apiVersion: dataprotection.kubeblocks.io/v1alpha1
kind: BackupRepo
metadata:
  name: my-repo
  annotations:
    dataprotection.kubeblocks.io/is-default-repo: "true"
spec:
  storageProviderRef: s3
  accessMethod: Tool
  pvReclaimPolicy: Retain
  volumeCapacity: 100Gi
  config:
    bucket: test-kb-backup
    endpoint: ""
    mountOptions: --memory-limit 1000 --dir-mode 0777 --file-mode 0666
    region: cn-northwest-1
  credential:
    name: s3-credential-for-backuprepo
    namespace: kb-system
  pathPrefix: ""
EOF

```

**Apply the BackupRepo:**

```bash
kubectl apply -f backuprepo.yaml
```

**Verify the BackupRepo:**

```bash
kubectl get backuprepo
```

For more details on configuring and using BackupRepo, refer to the [official BackupRepo documentation](https://kubeblocks.io/docs/release-0_9/user_docs/maintenance/backup-and-restore/backup/backup-repo).

### Using BackupRepo for Database Backups

Once a `BackupRepo` is created, you can reference it in your `BackupPolicy` to create a backup.

For more details on configuring and using BackupPolicy, refer to the [official BackupPolicy documentation](https://kubeblocks.io/docs/release-0_9/user_docs/maintenance/backup-and-restore/backup/configure-backuppolicy).

## Contributing

Contributions are welcome! If you want to add support for a new storage provider:

1. Create a new YAML file in the `storageprovider/` directory
2. Define the `StorageProvider` CR with appropriate templates and parameter schema
3. Test the provider with a BackupRepo
4. Submit a pull request

## References

- [KubeBlocks Official Documentation](https://kubeblocks.io/)
- [StorageProvider API Reference](https://kubeblocks.io/docs/release-0_9/user_docs/developer/api-reference/backup#dataprotection.kubeblocks.io/v1alpha1.StorageProvider)
- [BackupRepo Configuration Guide](https://kubeblocks.io/docs/release-0_9/user_docs/maintenance/backup-and-restore/backup/backup-repo)
- [KubeBlocks Backup and Restore](https://kubeblocks.io/docs/release-0_9/user_docs/maintenance/backup-and-restore/introduction)