# Terminology

[back](../README.md)

- [Locations](https://velero.io/docs/v1.15/locations/)
  - [BackupStorageLocation](https://velero.io/docs/v1.15/api-types/backupstoragelocation/): Velero backups location
    - Each Velero backup has one `BackupStorageLocation`, and one `VolumeSnapshotLocation` per volume provider
  - [VolumeSnapshotLocation](https://velero.io/docs/v1.15/api-types/volumesnapshotlocation/): Velero persistent volume snapshots location for associated backups
    - AWS and Azure do not allow you to create a volume snapshot in a different region than where the volume is
    - If you have multiple `VolumeSnapshotLocations` configured for a provider, you must always specify a valid `VolumeSnapshotLocation` when creating a backup, even if you are using File System Backup for volume backups. You can optionally decide to set the `--default-volume-snapshot-locations` flag using the velero server, which lists the default
    - if there’s only one possible volume snapshot location configured for each providers, Velero doesn’t require them to be explicitly specified when creating the backup
- Repository: File system backup location and configuration (uses restic, kopia).
- Snapshot: EBS volume snapshot
- FSB (File System Backup): With kopia created fodler backup and store in kopia subfoler in S3

Velero backs up Persistent Volume Claims (PVCs) in Kubernetes by taking snapshots or backing up the underlying storage, depending on the type of storage provisioner and the configuration.

**Here's a breakdown of how Velero handles PVC backups:**

_Volume Snapshots:_

- For supported cloud providers or storage backends (like AWS EBS, Azure Disks, GCP PD, etc.), Velero leverages the native snapshot capabilities of the storage provider.
- Velero, along with a Volume Snapshotter plugin, initiates a snapshot at the volume level. This snapshot is handled by the underlying cloud infrastructure and is efficient because it only captures the disk blocks (not individual files), providing a point-in-time snapshot.

_Restic Backup (File-Level Backup):_

- If you're using a storage provisioner that does not support native snapshots, or if you want a more flexible approach, Velero can back up the PVC contents using Restic.
- Restic works by copying the files inside the PVC to a backup repository (typically an object storage bucket).
- Unlike volume snapshots, which are block-level, Restic performs a file-level backup, capturing the actual files stored in the PVC. This can take longer, especially for large datasets, but provides an alternative for unsupported volumes.

**Restoring PVCs with Velero:**

During the restore process:

Volume Snapshots restore the PVC directly by re-attaching the snapshot, making the process relatively fast.
Restic Backups restore files individually back into a new PVC, which can take longer but is more granular.
Whether Velero takes snapshots or copies files depends on the PVC's storage class and whether Velero’s Volume Snapshotter or Restic is used.
