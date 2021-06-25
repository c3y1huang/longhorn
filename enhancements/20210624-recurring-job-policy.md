# Recurring Job Policy

## Summary

Allow users to create recurring backup/snapshot job policies with new `RecurringJobPolicy` CRD and have a new recurring job type - `Policy` that allows users to reference the policy in volume. 

Users can also update the recurring job policy, and Longhorn will automatically update the cronjob associated with volume and the policy.

When instruct Longhorn to delete a recurring job policy, this will also remove the cronjob associated with volume and the policy.

### Related Issues

https://github.com/longhorn/longhorn/issues/467

## Motivation

### Goals

- Ability to create multiple recurring backup/snapshot job policies with UI or YAML.
- Ability to apply volume recurring job type - `Policy` for policy reference.
- Ability to automatically apply default recurring backup/snapshot policy on the new volume.
- Recurring job policy should include recurring job settings; `name`, `type`, `cron`, `retain`.

### Non-goals [optional]

When volumes are all using the same recurring backup/snapshot policy, means all volumes will perform backups at the same time. As the original issue suggested, `Ideally, a range should be provided.`. This is currently an existing issue and should also be a problem when the recurring jobs are set at the storage class level.

> This will not be looked at here, reference to https://github.com/longhorn/longhorn/pull/2737#discussion_r660040985

## Proposal

#### Story 1 - refer volume to recurring job policy
As a Longhorn user / System administrator.

I want the ability to create a backup/snapshot recurring job as policy and select as type `Policy` during creating `Recurring Snapshot and Backup Schedule` or using YAML.

So I do not need to repeat the same recurring job setup for every volume.

#### Story 2 - default recurring job policy reference
As a Longhorn user / System administrator.

I want the ability to set backup/snapshot policy as default. New volume should automatically refer to the default backup/snapshot policy.

So I do not need to manually create the same recurring job for all volumes.


### User Experience In Detail

#### Story 1 - refer volume to recurring job policy

1. Create a recurring job policy in UI or `kubectl`.
2. In UI, Navigate to Volume, `Recurring Snapshot and Backup Schedule`
3. Select type as `Policy` from the `Type` list.
4. Select existing recurring job policy from the `Name` list.
5. Click `Save` should see `Schedule`, `Retain` referred from the `recurring job policy` CRs.

**Before enhancement**
Manually create recurring jobs for each volume.

**After enhancement**
Create a recurring job with type `Policy` automatically reference to the recurring job policy.

#### Story 2 - default recurring job policy reference

1. Create single or multiple recurring job policies with `default` set to `true` in UI or `kubectl`.
2. Create any new volume from UI or PVC with no recurring job setup in StorageClass, it should automatically apply the default recurring job policies.

> Question: Should the default recurring job policy overrule the storageClass recurring backup jobs? Or should they co-exist?


### API changes

> TBU.

## Design

### Implementation Overview

- Add recurring job policy CRD.
  - Update the ClusterRole to include `recurringjobpolices`.
  - The volume recurring job name will reference the recurring job policy name. It should be 8 characters or less. If the policy is named with more than 8 characters, an error shows on UI. This can be validated at new recurring job policy creation with specifying validation in the CRD YAML.
  - The `Type` should be one of `backup`, `snapshot` and can be validated in the CRD YAML pattern regex match.
  - Printer colume should include `Name`, `Type`, `Default`, `Recurring`, `Retain`, `Age`.

- In the volume controller,
  1. At volume creation, add default recurring job policies to volume `spec.recurringJobs` if volume `spec.recurringJobs` is an empty list.
  2. Skip `CreateVolumeCronJob` and `UpdateVolumeCronJob` the generated `cronJob.Spec.Schedule` is empty.

- Add recurring job policy controller.
  - The code structure will be the same as other controllers.
  - The controller will be informed by `recurringJobPolicyInformer`, `volumeInformer`.
  - Add the finalizer to the recurring job policy CRs.
  - Loop through volumes to scan for `spec.recurringJobs` with `task` set to `policy` and create and update the CronJobs.
    1. Generate a new cronjob object from the volume spec with an extra new label `longhorn.io/job-policy`.
      ```
      longhorn.io/job-policy: test1
      longhorn.io/job-task: policy
      ```
    2. set annotation `last-applied-cronjob-spec` and create cronjob if not already applied in cronjob.
    3. update annotation `last-applied-cronjob-spec` and update cronjob if the new cronjob is different than the last applied spec.
  - Clean up cronjobs when a recurring job policy gets deleted.
    1. Delete the cronjob with selected labels: `longhornvolume`, `longhorn.io/job-task`, `longhorn.io/job-policy`.
    2. Remove the finalizer when all the policy-associated cronjobs are deleted.

### Test plan

> WIP

#### Integration test - test_recurring_job_policy_volume_backup_reference

> TBU

#### Integration test - test_recurring_job_policy_volume_snapshot_reference

> TBU

#### Integration test - test_recurring_job_policy_default_backup_reference

> TBU

#### Integration test - test_recurring_job_policy_default_snapshot_reference

> TBU

#### Integration test - test_recurring_job_policy_backup_update

> TBU

#### Integration test - test_recurring_job_policy_snapshot_update

> TBU

### Upgrade strategy
There is no upgrade needed.

## Note [optional]
`None`
