# Helm Chart for DokuWiki Backup

This Helm chart sets up a simple backup strategy for a [Bitnami DokuWiki](https://github.com/bitnami/charts/tree/main/bitnami/dokuwiki/#installing-the-chart) deployment. DokuWiki does not use a database, and instead stores information using a file-based storage system. This chart sets up a backup strategy that integrates closely with the [BackupTool for Dokuwiki Plugin](https://www.dokuwiki.org/plugin:backup) to create and store backups in a similar manner.

This chart includes a backup container, scheduled cron jobs for daily, weekly, and monthly backups, and configurations for persistent storage and service accounts.

After setting up the chart, you can view the latest backups in the DokuWiki admin panel under the "Backup" tab. Additional backups are stored in the specified backup directory on the backup container.

## Configuration

In order to deploy this chart, you will need to have a Kubernetes cluster with the [Helm](https://helm.sh/) package manager installed.

Note: Your DokuWiki PVC must be of type ReadWriteMany in order for the backup container to be able to access it.

### Backup Container

This chart includes a backup container that provides an accessible storage location for your backups. If you do not require regular access to your backups, you can simply remove the backup container from the chart and the cron jobs will still function. You can also spin down the backup container if you do not need it running all the time. This will not impact the execution of the cron job backups.

- **backupcontainer.name**: Name of the backup container. Default is `backup-box`.
- **backupcontainer.image**: Docker image to use for the backup container. Default is `busybox:latest`.
- **backupcontainer.command**: Command to run within the container. Creates the file necessary to run healthchecks. Default is `[
  "/bin/sh",
  "-c",
  "touch /tmp/healthy && while true; do sleep 3600; done",
]`.
- **backupcontainer.resources**: Resource requests and limits for the container.

```yaml
backupcontainer:
  name: backup-box
  image: busybox:latest
  command:
    ["/bin/sh", "-c", "touch /tmp/healthy && while true; do sleep 3600; done"]
  resources:
    requests:
      memory: "64Mi"
      cpu: "50m"
    limits:
      memory: "128Mi"
      cpu: "100m"
```

### Shared Configuration for Cron Jobs

All cronjobs added to the chart will share the same base configuration. This configuration includes the image to use for the cron job, the command to run in the cron job container, the destination backup directory, the base name for the backup files, and the directory containing the files to be backed up.

- **sharedCronjobConfig.image**: Docker image to use for the cron job. Default is `bitnami/dokuwiki`.
- **sharedCronjobConfig.command**: Command to run in the cron job container. Default is `["/bin/bash", "-c"]`.
- **sharedCronjobConfig.destinationBackupDirectory**: Base directory for storing backups. Default is `/destination/backups`.
- **sharedCronjobConfig.backupName**: Base name for the backup files. Default is `dw-backup`.
- **sharedCronjobConfig.backupDirectory**: Directory containing the files to be backed up. Default is `/bitnami/dokuwiki/data/media/wiki/backup`.

```yaml
sharedCronjobConfig:
  image: bitnami/dokuwiki
  command: ["/bin/bash", "-c"]
  destinationBackupDirectory: /destination/backups
  backupName: dw-backup
  backupDirectory: /bitnami/dokuwiki/data/media/wiki/backup
```

### Cron Jobs for Scheduling Backups

Add named configurations to generate additional cron jobs. Cronjobs will store their backups in a named directory in the backup container based on the name of the cronjob. When run, they will create a new backup file and store it in the specified backup directory, and will remove any old backup files that are older than the `daysToRetain` parameter.

- **cronjobs.\*.enabled**: Enable the backup cron job.
- **cronjobs.\*.schedule**: Schedule to run the backup cron job.
- **cronjobs.\*.successfulJobsHistoryLimit**: Number of successful jobs to retain.
- **cronjobs.\*.failedJobsHistoryLimit**: Number of failed jobs to retain.
- **cronjobs.\*.daysToRetain**: Number of days to retain backups created under this cronjob.

```yaml
cronjobs:
  - name: daily
    enabled: true
    schedule: "0 0 * * *" # Schedule to run daily at midnight
    successfulJobsHistoryLimit: 3
    failedJobsHistoryLimit: 3
    daysToRetain: 7
  - name: weekly
    enabled: true
    schedule: "0 4 * * 1" # Schedule to run weekly on Mondays at 4am
    successfulJobsHistoryLimit: 3
    failedJobsHistoryLimit: 3
    daysToRetain: 30
  - name: monthly
    enabled: true
    schedule: "0 6 1 * *" # Schedule to run monthly on the 1st at 6am
    successfulJobsHistoryLimit: 3
    failedJobsHistoryLimit: 3
    daysToRetain: 365
```

### Test Job Configuration

The test job is used to verify that the backup container is working correctly. It uses the configuration information of the first cronjob listed. It will create a new backup file and store it in the specified backup directory. This job is not enabled by default, but you can enable it by setting the `testJob.enabled` parameter to `true`.

- **testJob.enabled**: Enable the test backup job. Default is `false`.
- **testJob.retentionTime**: Time (in seconds) to retain the test job. Default is `86400` (one day).

```yaml
testJob:
  enabled: false
  retentionTime: 86400
```

### Persistent Volume Claim (PVC) Configuration

This chart includes a persistent volume claim (PVC) that provides an accessible storage location for your backups. It is recommended to create a PVC independent of the chart, and to use a storage class that is compatible with the backup container.

- **pvc.existingClaim**: Whether an existing claim is being used. Default is `true`.
- **pvc.name**: Name of the backup container PVC. Default is `backup-pvc`.
- **pvc.storageClass**: Storage class for the PVC. Should be a file backup storage class. Default is `netapp-file-backup`.
- **pvc.accessModes**: Access mode for the PVC. Default is `ReadWriteMany`.
- **pvc.storage**: Requested storage size for the backup container PVC. Default is `200Mi`.

```yaml
pvc:
  existingClaim: true
  name: backup-pvc
  storageClass: netapp-file-backup
  storage: 1Gi
```

### Service Account Configuration

- **serviceAccount.existingServiceAccount**: Whether an existing service account is being used. Default is `false`.
- **serviceAccount.exists**: Whether the service account exists. Default is `false`.
- **serviceAccount.name**: Name of the service account. Default is `backup-sa`.

```yaml
serviceAccount:
  existingServiceAccount: false
  name: backup-sa
```

### Volume Configuration

Enter the name of the volume your current DokuWiki instance is using.

- **volumes.dokuwikiData.claimName**: Name of the persistent volume claim for the DokuWiki instance.

```yaml
volumes:
  dokuwikiData:
    claimName: dokuwiki-pvc-name
```

## Usage

To deploy the Helm chart with the default values, run:

```sh
helm install my-dokuwiki-backup ./dokuwiki-backup
```

To customize the deployment, create a `values.yaml` file with your desired configurations and run:

```sh
helm install my-dokuwiki-backup ./dokuwiki-backup -f values.yaml
```

This will set up the backup container and the cron jobs according to your specified configuration, ensuring that your DokuWiki data is backed up regularly.
