# Backing up and Restoring a k0rdent Management Cluster

Any production system needs to provide Disaster Recovery features, and the heart of these capabilities is the ability to
perform backup and restore operations. In this chapter we'll look at backing up and restoring k0rdent management cluster
so that in case of an emergency, you can restore your system to its previous condition.

While it's possible to back up a Kubernetes cluster manually, it's better to build on the work
of others. In this case we're going to leverage the [`velero`](https://velero.io/) project for backup management
on the backend and see how it integrates with k0rdent to ensure data persistence and recovery.

## Motivation

The primary goal of this feature is to provide a reliable and efficient way to back up and restore
a k0rdent deployment in the event of a disaster that impacts the management cluster.
By using `velero` as the backup provider, we can create consistent backups across
different cloud storage while maintaining the integrity of critical resources.

The main goal of the feature is to provide:

- Backup: The ability to backup all configuration objects created and managed by k0rdent, including
  into an offsite location.
- Restore: The ability to create configuration objects from a specific Backup in order to create a management
  cluster in the same state that existed at the time of backup.
- Disaster Recovery: The ability to restore k0rdent on another management cluster
  using restore capability, plus ensuring that clusters are not recreated or lost.
- Rollback: the possibility to manually restore after a specific event, such as a failed k0rdent upgrade

## Velero as Provider for Backups

[`Velero`](https://velero.io/) is an open-source tool that simplifies backing up and restoring clusters as well as individual resources. It seamlessly integrates into k0rdent management environment to provide robust disaster recovery capabilities.

The `velero` instance is part of the Helm chart that installs k0rdent, which means that it
can be customized if necessary. (You can find information on customizing Velero [below](#customization))

k0rdent manages the schedule and is responsible for collecting data to be included in a backup.

## Scheduled Backups

Backups should be scheduled on a regular basis, depending on how often information changes.

### Preparation

Before you create a scheduled backup, you need to perform a few preparatory steps:

1. Prepare a cloud storage location, such as an Amazon S3 bucket, to which to save backups.
2. Create a [`BackupStorageLocation`](https://velero.io/docs/v1.15/api-types/backupstoragelocation/)
   object referencing a `Secret` with credentials to access the cloud storage
   (if the multiple credentials feature is supported by the plugin).

   For example, if you are using Amazon S3, your `BackupStorageLocation` and the related `Secret` might look like this:

   ```yaml
   ---
   apiVersion: v1
   data:
     # base64-encoded credentials for Amazon S3 in the following format:
     # [default]
     # aws_access_key_id = EXAMPLE_ACCESS_KEY_ID
     # aws_secret_access_key = EXAMPLE_SECRET_ACCESS_KEY
          cloud: W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkID0gRVhBTVBMRV9BQ0NFU1NfS0VZX0lECmF3c19zZWNyZXRfYWNjZXNzX2tleSA9IEVYQU1QTEVfU0VDUkVUX0FDQ0VTU19LRVkKICA=
   kind: Secret
   metadata:
     name: cloud-credentials
     namespace: kcm-system
   type: Opaque
   ---
   apiVersion: velero.io/v1
   kind: BackupStorageLocation
   metadata:
     name: aws-s3
     namespace: kcm-system
   spec:
     config:
       region: <your-region-name>
     default: true # optional, if not set, then storage location name must always be set in ManagementBackup
     objectStorage:
       bucket: <your-bucket-name>
     provider: aws
     backupSyncPeriod: 1m
     credential:
       name: cloud-credentials
       key: cloud
   ```

   You can get more information how to build these objects at the [official Velero documentation](https://velero.io/docs/v1.15/locations).

### Create Backup

Periodic backups are handled by a `ManagementBackup` object, which uses a [Cron](https://en.wikipedia.org/wiki/Cron) expression
for its `.spec.schedule` field. 
If you don't set the `.spec.schedule` field, Velero will instead work with  [backup on demand](#backup-on-demand).

Optionally, set the name of the `BackupStorageLocation` `.spec.backup.storageLocation`.
The default location is the `BackupStorageLocation` object with `.spec.default` set to `true`.

For example, you can create a `ManagementBackup` object that backs up to the storage object created in the previous step
every 6 minutes would look like this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ManagementBackup
metadata:
  name: kcm
spec:
  schedule: "0 */6 * * *"
  storageLocation: aws-s3
```

## Backup on Demand

To create a single backup of the existing k0rdent managagement cluster informaiton, you can create a `ManagementBackup` object
using a YAML document and the `kubectl` CLI. The object then creates only one instance of a backup. For example you can backup
to the location created earlier:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ManagementBackup
metadata:
  name: example-backup
spec:
  storageLocation: aws-s3
```

## What's Included in the Backup

The backup includes all of k0rdent `kcm` component resources, parts of the `cert-manager`
components required for other components creation, and all the required resources
of `CAPI` and the `ClusterDeployment`s currently in use in the management cluster.

> EXAMPLE: An example set of labels, and objects satisfying these labels will
> be included in the backup:
>
>```text
> cluster.x-k8s.io/cluster-name="cluster-deployment-name"
> cluster.x-k8s.io/provider="bootstrap-k0sproject-k0smotron"
> cluster.x-k8s.io/provider="cluster-api"
> cluster.x-k8s.io/provider="control-plane-k0sproject-k0smotron"
> cluster.x-k8s.io/provider="infrastructure-aws"
> controller.cert-manager.io/fao="true"
> helm.toolkit.fluxcd.io/name="cluster-deployment-name"
> k0rdent.mirantis.com/component="kcm"
>```

## Restoration

> NOTE: Caveats and limitations
>
> Please refer to the
> [official migration documentation](https://velero.io/docs/v1.15/migration-case/#before-migrating-your-cluster)
> to familiarize yourself with potential limitations of the Velero backup system.

In the event of disaster, you can restore from a backup by doing the following:

1. Create a clean k0rdent installation. Specifically, you want to **avoid** creating a `Management` object
   and similiar objects because they will be part of your restored cluster. You can remove these objects after
   installation, but you can also install k0rdent without them in the first place:

    ```bash
    helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm \
     --version <version> \
     --create-namespace \
     --namespace kcm-system \
     --set controller.createManagement=false \
     --set controller.createAccessManagement=false \
     --set controller.createRelease=false \
     --set controller.createTemplates=false
    ```

2. Create the `BackupStorageLocation`/`Secret` objects that were
   created during the [preparation stage](#preparation) of creating a backup (preferably the same depending on a plugin).
3. Restore k0rdent system. To do this you have two options. The first is to directly create the [`Restore`](https://velero.io/docs/v1.15/api-types/restore/) object:

   ```yaml
    apiVersion: velero.io/v1
    kind: Restore
    metadata:
      name: <restore-name>
      namespace: kcm-system
    spec:
      backupName: <backup-name>
      excludedResources:
      # the following are velero defaults, so it is recommended to keep them
      - nodes
      - events
      - events.events.k8s.io
      - backups.velero.io
      - restores.velero.io
      - resticrepositories.velero.io
      - csinodes.storage.k8s.io
      - volumeattachments.storage.k8s.io
      - backuprepositories.velero.io
      existingResourcePolicy: update
      includedNamespaces:
      - '*'
    ```

    You can also restore from backup using the [velero CLI](https://velero.io/docs/v1.15/basic-install/#install-the-cli):

    ```shell
    velero --namespace kcm-system restore create <restore-name> --existing-resource-policy update --from-backup <backup-name>
    ```

4. This, of course, will take some time.  Wait until the `Restore` status is `Completed` and all `kcm` components are up and running.

   ```shell
   kubectl get Restore <restore-name> -n kcm-system
   ```


<!-- ## Upgrades and rollback

TODO:  Show the command for seeing the Retore ojbect

TODO: fill out this section when the upgrade backup is implemented

Describe that the backup is created each time before the `kcm` upgrade.

Refer to how to rollback properly (probably just follow the restoration part)

TBD, no implementation exists yet -->

## Caveats / Limitation

<!-- TODO: not sure whether it is okay to mention that explicitly since we could implement
it somewhere in the future utilizing velero hooks -->

All `velero` caveats and limitations are transitively implied in the `k0rdent`.

In particular, that means no backup encryption is provided until it is implemented
by a `velero` plugin that supports both encryption and cloud storage backups.

# Customization

This section covers different topics of customization regarding backing up and restoring k0rdent.

## Velero installation

The Velero helm chart is supplied with the
[k0rdent helm chart](https://vmware-tanzu.github.io/helm-charts/)
and is enabled by default.
You have two choices for customizing the chart values.

1. If installing using `helm`, you can add corresponding parameters to the `helm install` command.
   For example:

    ```bash
    helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm \
     --version <version> \
     --create-namespace \
     --namespace kcm-system \
     --set velero.initContainers[0].name=velero-plugin-for-<PROVIDER NAME> \
     --set velero.initContainers[0].image=velero/velero-plugin-for-<PROVIDER NAME>:<PROVIDER PLUGIN TAG> \
     --set velero.initContainers[0].volumeMounts[0].mountPath=/target \
     --set velero.initContainers[0].volumeMounts[0].name=plugins
    ```

2. Once Velero is installed, you can also create or modify the existing `Management` object in the `.spec.config.kcm`, for example:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: Management
    metadata:
      name: kcm
    spec:
      core:
        kcm:
          config:
            velero:
             initContainers:
               - name: velero-plugin-for-<PROVIDER NAME>
                 image: velero/velero-plugin-for-<PROVIDER NAME>:<PROVIDER PLUGIN TAG>
                 imagePullPolicy: IfNotPresent
                 volumeMounts:
                   - mountPath: /target
                     name: plugins
          # ...
    ```

To fully disable `velero`, set the `velero.enabled` parameter to `false`.

## Schedule expression format

The `ManagementBackup` `.spec.schedule` field accepts a correct
[Cron](https://en.wikipedia.org/wiki/Cron) expression,
along with the
[nonstandard predefined scheduling definitions](https://en.wikipedia.org/wiki/Cron#Nonstandard_predefined_scheduling_definitions)
and an extra definition `@every` with a number and a valid time unit
(valid time units are `ns, us (or µs), ms, s, m, h`).

The following list contains `.spec.schedule` acceptable example values:

- `0 */1 * * *` (standard Cron expression)
- `@hourly` (nonstandard predefined definition)
- `@every 1h` (extra definition)

## Putting extra objects in a Backup

If you need to back up objects other than those [backed up by default](overview.md#whats-included-in-the-backup),
you can add the label `k0rdent.mirantis.com/component="kcm"` to these objects.

All objects containing the label will be automatically added to the backup.