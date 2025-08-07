# Backup / Restore

[back](../README.md)

:warning: Velero can do one type of backup at a time: FSB backup or snapshot backup. Both can not be combined

## Manifest only backup

Data does not backup, only deployment and PV/PVC could be restored

See [MiniO examples](https://velero.io/docs/main/contributions/minio/)

```bash
# deployment
kubectl apply -f examples/nginx-app/no-pv.yaml

# backup
velero backup create no-pv --include-namespaces no-pv --storage-location minio
# or
velero backup create no-pv --selector app=nginx
```

## Manifest, FSB, PV/PVC backup

Create sample app and generate data

```bash
# create sc,pv and install deployment
kubectl apply -f examples/nginx-app/hostpath-pv.yaml

# genearate extra data
POD=$(kubectl get pods -n hostpath-pv -l app=nginx -o json | jq -r .items[0].metadata.name)
kubectl -n pods -n hostpath-pv exec $POD -c nginx -- ls /var/log/nginx
kubectl -n pods -n hostpath-pv exec $POD -c nginx -- bash -c 'df -h > /var/log/nginx/df.txt'
kubectl -n pods -n hostpath-pv exec $POD -c nginx -- cat /var/log/nginx/df.txt
```

### [Filesystem Backup (FSB)](https://velero.io/docs/v1.15/file-system-backup)

Use one of the way for FSB backup:

- Add annotations to deployment like below:

    ```yaml
    backup.velero.io/backup-volumes: nginx-logs,pvc-volume,emptydir-volume  # when `--default-volumes-to-fs-backup` NOT used
    backup.velero.io/backup-volumes-excludes: pvc1-vm  # when `--default-volumes-to-fs-backup` used
    ```

- or Use `--default-volumes-to-fs-backup` arg to backup all volumes with FSB

```bash
# if annotation not added
velero backup create hostpath-pv --include-namespaces hostpath-pv --storage-location minio --default-volumes-to-fs-backup

# if annotation added
velero backup create hostpath-pv --include-namespaces hostpath-pv --storage-location minio --default-volumes-to-fs-backup
```

### [Snapshot backup](https://velero.io/docs/v1.15/csi)

There are two ways Velero can create backup with CSI snapshot

1. Without moving snapshot data to backup storage: Velero backs-up K8s resources to backup storage but do not move data to backup storage. Volume snapshot is created but data is kept in snapshot itself. Some storage provider keeps PV and Volume snapshot in same storage. So, If storage itself is lost, PV along with Snapshot data data will be lost and cannot be restored.
2. With moving snapshot data to backup storage: Velero backs-up K8s resources along with snapshot data to backup storage. A volume snapshot will be created for the PVs and the data from snapshot is moved to backup storage using Kopia.

#### 1. Create snapshot backup (without moving snapshot data to backup storage)

**Default Behavior**: You can simply create a VolumeSnapshotClass for a particular driver and put a label on it to indicate that it is the default VolumeSnapshotClass for that driver. **Note:** For each driver type, there should **only be 1** VolumeSnapshotClass with the label

Velero will include the generated VolumeSnapshot and VolumeSnapshotContent objects in the backup tarball, as well as upload all VolumeSnapshots and VolumeSnapshotContents objects in a JSON file to the object storage system. Note that **only** Kubernetes objects are uploaded to the object storage, **not** the data in snapshots.

```bash
# Define which snapshotclass will be used by adding label
kubectl label volumeSnapshotclass csi-hostpath-snapclass velero.io/csi-volumesnapshot-class=true
# Change deletionPolicy: Delete -> Retain  if you want to preserve volumes
# kubectl patch volumesnapshotclasses csi-hostpath-snapclass --type='json' -p '[{"op": "replace", "path": "/deletionPolicy", "value": "Retain"}]'

# hostpath external-snapshotter
velero backup create hostpath-pv --include-namespaces hostpath-pv --storage-location minio

# AWS
velero backup create nginx-backup-pv --include-namespaces nginx-example-pv --storage-location minio --volume-snapshot-locations ebs-us-east-1
```

With above command also monitor snapshot section in ec2 service of aws.

#### 2. Create snapshot data movement backup (with [moving snapshot data to backup storage](https://velero.io/docs/v1.15/csi-snapshot-data-movement))

- [BackupPVC Configuration for Data Movement Backup](https://velero.io/docs/v1.15/data-movement-backup-pvc-configuration/)

```bash
# MiniO + hostPath
velero backup create hostpath-pv-move-fs --include-namespaces hostpath-pv --storage-location minio --snapshot-move-data

# AWS
velero backup create nginx-backup-pv --include-namespaces nginx-example-pv  --snapshot-move-data \
  --storage-location s3 --volume-snapshot-locations ebs-us-east-1
```

Velero backs-up Kubernetes resources and PV’s data into a backup storage location like AWS S3 bucket, or any cloud object storage. It uses restic or Kopia (Starting from velero 1.10) tool to upload data inside a persistent volume to object storage.

## Verify backup

```bash
# check backup status
mc ls minio/alyvusal-terraform-backend/velero/backups/hostpath-pv
mc ls minio/alyvusal-terraform-backend/velero/kopia/hostpath-pv

velero get backups
velero describe backups hostpath-pv
velero describe backups hostpath-pv --details

velero backup logs no-pv
velero backup logs no-pv | grep Error

kubectl -n velero get podvolumebackups
kubectl -n hostpath-pv get volumesnapshotcontent
kubectl -n nginx-example-pv get volumesnapshotclasses

# verify data movement
kubectl -n velero get datauploads
```

## Restore

```bash
# restore
velero restore create --from-backup no-pv
# restore to different ns
velero restore create --from-backup hostpath-pv --namespace-mappings hostpath-pv:hostpath-fs

velero get restores
kubectl -n velero get podvolumerestores.velero.io

POD=$(kubectl get pods -n hostpath-pv -l app=nginx -o json | jq -r .items[0].metadata.name)
kubectl -n pods -n hostpath-pv exec $POD -c nginx -- ls /var/log/nginx
kubectl -n pods -n hostpath-pv exec $POD -c nginx -- cat /var/log/nginx/df.txt
```

## Delete

```bash
# delete
velero backup delete no-pv
kubectl delete -f examples/nginx-app/hostpath-pv.yaml
```

### Restore to different cluster

```bash
# to restore to different cluster, install velero with same options and restore
# wait some seconds if backups not seen by velero
kubectl config use-context kind-dev
velero get backups
velero restore create --from-backup nginx-backup

velero get restores
kubectl -n velero get podvolumerestores
```

## Pay attention to

- By default, velero install does not enable the use of File System Backup (FSB) to take backups of all pod volumes. You must apply an annotation to every pod which contains volumes for Velero to use FSB for the backup.
- In supported provider not found in [list](https://velero.io/docs/main/supported-providers/) then check [File System Backup](https://velero.io/docs/main/file-system-backup/) document. `hostPath` volumes are not supported
- To backup pod folder add annotations like: `backup.velero.io/backup-volumes: nginx-logs`

## [Scheduled backup](https://velero.io/docs/main/disaster-case/)

```bash
# include one NS other wise backups all
velero schedule create aa --schedule "* * * * *" --include-namespaces=nginx-example

kubectl get -n velero schedules.velero.io
kubectl get -n velero schedules.velero.io -o yaml
```

## Kopia

```bash
# Put trailing slash (/) if you want to use prefix as directory (It'i a directory in S3/MiniO)
kopia repository connect s3 \
  --bucket=alyvusal-terraform-backend \
  --prefix=velero/kopia/hostpath-pv/ \
  --access-key=admin     \
  --secret-access-key=admin123 \
  --region=minio \
  --endpoint=minio-api-192.168.0.100.nip.io \
  --disable-tls-verification \
  -p $(kubectl -n velero get secret velero-repo-credentials -o json | jq -r '.data."repository-password"' | base64 -d)

mkdir ~/tmp/nginx-logs
cd ~/tmp/nginx-logs
kopia snapshot list --all
kopia snapshot restore <snapshot-id> --target-path /tmp/nginx-logs

kopia content show k785b54dc690c33e97d711b015c935e99
kopia snapshot restore k785b54dc690c33e97d711b015c935e99 .
```

## [Examples](https://velero.io/docs/main/examples/)

Basic example (without PersistentVolumes)

```bash
# install app
kubectl apply -f examples/nginx-app/base.yaml

# Create a backup:
velero backup create nginx-backup --include-namespaces nginx-example \
  --storage-location s3 --volume-snapshot-locations ebs-us-east-1

# Simulate a disaster:
kubectl delete namespaces nginx-example

# Wait for the namespace to be deleted.
# Restore your lost resources:
velero restore create --from-backup nginx-backup
```

Snapshot example (with PersistentVolumes)

NOTE: For Azure, you must run Kubernetes version 1.7.2 or later to support PV snapshotting of managed disks.

```bash
# Start the sample nginx app:
kubectl apply -f examples/nginx-app/with-pv-aws-1.yaml

# Create a backup with PV snapshotting. --csi-snapshot-timeout is used to setup time to wait before CSI snapshot creation timeout. The default value is 10 minutes:

velero backup create nginx-backup-pv --include-namespaces nginx-example-pv \
  --storage-location s3 --volume-snapshot-locations ebs-us-east-1

# check backup
velero describe backups

# see details for filesystem backup
velero describe backups --details
kubectl -n velero get podvolumebackups

# Simulate a disaster:
kubectl delete namespaces nginx-example-pv

# Because the default reclaim policy for dynamically-provisioned PVs is “Delete”, these commands should trigger your cloud provider to delete the disk that backs the PV. Deletion is asynchronous, so this may take some time. Before continuing to the next step, check your cloud provider to confirm that the disk no longer exists.

# Restore your lost resources:
velero restore create --from-backup nginx-backup-pv
```

## REFERENCE

- [Hooks](https://velero.io/docs/v1.5/backup-hooks/)
