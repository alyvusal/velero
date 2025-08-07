# Install with CLI

[back](../README.md)

See [custom namespace installation](https://velero.io/docs/v1.15/namespace/)

## Install each part separately

### 1. Install velero server alone

```bash
velero install \
  --no-default-backup-location \
  --no-secret \
  --use-volume-snapshots=false

kubectl get crd | grep velero

# dry-run
velero install \
  --no-default-backup-location \
  --no-secret \
  --use-volume-snapshots=false \
  --dry-run -o yaml
```

### 2. Add FS backup support

- :warning: It is said FS backup and snapshot backup **can not be combined**. FS abckup used where snapshotting is not supported like on NFS, EFS
- FS backup is read from PV directly and may not be consistent
- Use velero hook to freeze apps during backup
- `--default-volumes-to-fs-backup`: If you are planning to only use FSB for volume backups, you can run the velero install command with this flag
  - Alternatively, you can set the `--default-volumes-to-fs-backup` on an individual backup to to make sure Velero uses FSB for each volume being backed up

1. Install node agent (kopia, restic) to use for backup files in file system level:
   - If you’ve already run velero install without the `--use-node-agent` or `--privileged-node-agent` flag, you can run the same command again, including the `--use-node-agent` or `--privileged-node-agent` flag, to add CSI snapshot data movement to your existing install.

    ```bash
    velero install \
      --no-default-backup-location \
      --no-secret \
      --use-volume-snapshots=false \
      --use-node-agent
    ```

### 3. Add CSI Snapshotting Support

Recent CSI versions (not all, ie EFS CSI) also supports volume snapshotting. Since CSI is not part of k8s, it must be installed separately for SC and snapshotters.

For each CSI driver you need to have a `VolumeSnapshotClass` with dletion policy set to `Retain`

```bash
velero install \
  --no-default-backup-location \
  --no-secret \
  --use-volume-snapshots=false \
  --use-node-agent \
  --features=EnableCSI
```

### 4. Create Backup Locations

1. Add [plugins](https://github.com/orgs/vmware-tanzu/repositories?q=velero-plugin-for-) for clouds, here we will use AWS or MiniO

    ```bash
    # AWS integration
    velero plugin add velero/velero-plugin-for-aws:v1.11.1 --confirm
    ```

2. Create secret for AWS/Minio

    ```bash
    kubectl -n velero create secret generic aws-credentials --from-file=aws=/tmp/credentials-velero
    ```

3. Create backup storage location for metadata and filesystem backups

    - `--default`: make it default location
    - `--config region=us-east-1`: For AWS
    - `--config region=minio,s3ForcePathStyle="true",s3Url=http://minio-api-192.168.0.100.nip.io`: For [Minio](https://velero.io/docs/main/contributions/minio/)

    Please make sure to set parameter `s3ForcePathStyle=true`. The parameter is used to set the Velero integrated AWS SDK data query address style. There are two types of the address: [virtual-host and path-style](https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html). If the `s3ForcePathStyle=true` is not set, the default value is false, then the AWS SDK will query in virtual-host style, but the MinIO server only support path-style address by default. The miss match will mean Velero can upload data to MinIO, but cannot download from MinIO. This [link](https://github.com/vmware-tanzu/velero/issues/7268) is an example of this issue. It can be resolved by two ways:
    - Set `s3ForcePathStyle=true` for parameter `--backup-location-config` when installing Velero. This is the preferred way.
    - Make MinIO server support **virtual-host** style address. Add the `MINIO_DOMAIN` environment variable for MinIO server will do the magic.

    ```bash
    velero backup-location create minio \
      --bucket alyvusal-terraform-backend \
      --provider aws \
      --config region=minio,s3ForcePathStyle="true",s3Url=http://minio-api-192.168.0.100.nip.io \
      --prefix velero \
      --credential=aws-credentials=aws

    # verifuy backup locations
    velero backup-location get
    kubectl get -n velero backupstoragelocations.velero.io minio -o yaml
    velero get backup-locations --output yaml

    # modify secret for backup storage location
    velero backup-location set minio --credential=<secret-name>=<key-within-secret>

    # set/change default backup location
    velero backup-location set minio --default
    # velero server --default-backup-storage-location minio  # change on server pod

    # remove default backup location
    velero backup-location set minio --default=false
    ```

    `prefix=velero` is `velero` folder name in `alyvusal-terraform-backend` bucket.

4. Create snapshot location if snaphot option enabled

    - `--config region=us-east-1`: For AWS
    - `--config region=minio,s3ForcePathStyle="true",s3Url=http://minio-api-192.168.0.100.nip.io`: For [Minio](https://velero.io/docs/main/contributions/minio/)

    ```bash
    velero snapshot-location create ebs-us-east-1 \
      --provider aws \
      --config region=minio,s3ForcePathStyle="true",s3Url=http://minio-api-192.168.0.100.nip.io \
      --credential=aws-credentials=aws

    velero snapshot-location get
    kubectl get -n velero volumesnapshotlocations.velero.io

    # modify secret for snapshot storage location
    velero snapshot-location set ebs-us-east-1  --credential=<secret-name>=<key-within-secret>

    # Set default volume snapshot location
    # velero server --default-volume-snapshot-locations="<PROVIDER-NAME>:<LOCATION-NAME>,<PROVIDER2-NAME>:<LOCATION2-NAME>"
    velero server --default-volume-snapshot-locations="aws:s3"
    ```

## All in one installation

All-in-one installation (s3 integration, snapshot backup, fs backup)

- `--use-node-agent`: Enable FS backups (i.e. Restic, Kopia)
- `--privileged-node-agent`: For some use cases, Velero node-agent requires to run under privileged mode. For example, when backing up block volumes, it is required to allow the node-agent to access the block device. If node-agent is not running in a privileged mode, it will not be able to access pods volumes within the mounted hostpath directory because of the default enforced SELinux mode configured in the host system level. You can create a custom SCC to relax the security in your cluster so that node-agent pods are allowed to use the hostPath volume plugin without granting them access to the privileged SCC.
- `--use-volume-snapshots=true`: create snapshot location automatically, we will create snapshot-location later
- `--features=EnableCSI,EnableAPIGroupVersions`: enable features:
  - `[EnableCSI](https://velero.io/docs/main/csi)`: enable integration Velero with the CSI volume snapshot APIs
  - `[EnableAPIGroupVersions](https://velero.io/docs/main/enable-api-group-versions-feature/)`: enabling the support of multiple API group versions
  - Enabling and disabling feature flags will require modifying the Velero deployment and also the node-agent daemonset. This may be done from the CLI by uninstalling and re-installing Velero, or by editing the deploy/velero and daemonset/node-agent resources in-cluster.
- `--default-volumes-to-fs-backup`: If you are planning to only use FSB for volume backups, you can run the velero install command with this flag
- `--backup-location-config region=us-east-1`
- `--backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio-api-192.168.0.100.nip.io`: For [Minio](https://velero.io/docs/main/contributions/minio/)
- `--snapshot-location-config region=us-east-1`: For AWS
- `--snapshot-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio-api-192.168.0.100.nip.io`: For [Minio](https://velero.io/docs/main/contributions/minio/)

```bash
velero install \
  --features=EnableCSI \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.11.1 \
  --bucket alyvusal-terraform-backend \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio-api-192.168.0.100.nip.io \
  --prefix=velero \
  --use-node-agent \
  --secret-file /tmp/credentials-velero \
  --use-volume-snapshots=true \
  --snapshot-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio-api-192.168.0.100.nip.io
```

`prefix=velero` is `velero` folder name in `alyvusal-terraform-backend` bucket.

check exmaple [credentials-velero file](./credentials-velero) for content format

### Extra features

```bash
# Enable clint side features
velero client config set features=feature1,feature2...
```

Add following options to enable [features](https://github.com/vmware-tanzu/velero/blob/main/pkg/apis/velero/v1/constants.go):

Enabling and disabling feature flags will require modifying the Velero deployment and also the node-agent daemonset and add `EnableCSI` to `--features=` arg. This may be done from the CLI by uninstalling and re-installing Velero, or by editing the deploy/velero and daemonset/node-agent resources in-cluster.

```bash
kubectl -n velero edit deploy/velero
kubectl -n velero edit daemonset/node-agent
```

To generate the YAML without applying it to your cluster, use the `--dry-run -o yaml` flags.
