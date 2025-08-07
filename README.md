# Velero Backup

[back](../README.md)

## [Terminology](./docs/terminology.md)

Velero and K8up are designed to back up Kubernetes resources and Persistent Volumes (PVs). They do not natively support backing up arbitrary files from within a pod's filesystem unless those files are stored in a mounted Persistent Volume (PV).

If the files are not in a PV or PVC, you will need to rely on a custom script or tool (like AWS CLI, Restic, or another backup utility) to directly copy the pod files to S3.

## [Install](https://velero.io/docs/v1.15/basic-install/#option-2-github-release)

if you want to backup to minio then

```bash
# install CSI snapshotter for kind
../kubernetes/storage/hostpath-csi.sh install
../kubernetes/storage/cloudcasa-csi-checker.sh

# deploy minio
helm upgrade -i operator minio-operator/operator \
  --namespace minio-operator \
  --create-namespace \
  --version 7.0.0

helm upgrade -i tenant minio-operator/tenant \
  --namespace minio-operator \
  --create-namespace \
  --version 7.0.0 \
  -f ../minio/k8s/helm/tenant-values.yaml

# after tenants created
kubectl -n minio-operator get pods -l v1.min.io/tenant=minio
mc mb minio/alyvusal-terraform-backend

# create and note iam user secrets credentials in aws and add to /tmp/credentials-velero
cp ./credentials-velero /tmp/credentials-velero
```

### [with CLI](./docs/install-cli.md)

### [with HELM](./docs/install-helm.md)

## [Backup](./docs/backup.md)

## REFERENCE

- [Velero plugins for AWS](https://github.com/vmware-tanzu/velero-plugin-for-aws)
- API types: Backup, Restore, Schedule, BackupStorageLocation, VolumeSnapshotLocation
  - [Backup Storage Location](https://velero.io/docs/v0.11.0/api-types/backupstoragelocation/) and [github](https://github.com/vmware-tanzu/velero-plugin-for-aws/blob/main/backupstoragelocation.md)
  - [Volume Snapshot Location](https://velero.io/docs/main/api-types/volumesnapshotlocation/) and [github](https://github.com/vmware-tanzu/velero-plugin-for-aws/blob/main/volumesnapshotlocation.md)
    - When using FSB on a storage that doesn’t have Velero support for snapshots, the `--use-volume-snapshots=false` flag prevents an unused VolumeSnapshotLocation from being created on installation.
- [Vmware docs related to velero on Tanzu - great examples](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid-Integrated-Edition/1.20/tkgi/GUID-backup-and-restore-work.html)
- [Pod volume backup path annotation watched for prometheus](https://github.com/bitsbeats/velero-pvc-watcher)
- [FSB Monitoring](https://velero.io/docs/main/file-system-backup/#monitor-backup-annotation)
  - [Backup Kubernetes using Velero and CSI volume snapshot](https://medium.com/linux-shots/backup-kubernetes-using-velero-and-csi-volume-snapshot-4155d4e32e5d)
