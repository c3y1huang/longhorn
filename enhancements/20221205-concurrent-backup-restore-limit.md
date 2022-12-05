# Concurrent Backup Restore Limit

## Summary
Currently, Longhorn does not allow the user to set a boundary on the number of concurrent volume backup restoring.

Have a new `concurrent-backup-restore-limit` setting allows user to limit the concurring backup restoring. This can help to lower the potential risk of overloading cluster when volume requested to restore from backup at the same time.

### Related Issues
https://github.com/longhorn/longhorn/issues/4558

## Motivation
### Goals
Introduce a new `concurrent-backup-restore-limit` setting to allow the user to set a boundary to the number of concurrent volume backup restoring. 

### Non-goals
`None`

## Proposal
1. Introduce a new `concurrent-backup-restore-limit` setting.
1. Introduce a new `longhorn.io/backup-restore-priority` label
  - Tag Longhorn Engine with time in `Unix nano` when volume needs to restore from backup. This value is used to determine the priority of the volume backup restore.
  - Tag with `in-progress` during backup restore. This value will be used to check on the current number of restoring volumes.

### User Stories
Allow the user to set the concurrent backup restore limit to avoid potential cluster overload when Longhorn volume is restoring from backup at the same time.

### User Experience In Detail
1. Longhorn pends the backup restore when the number of `longhorn.io/backup-restore-priority: in-progress` reached the `concurrent-backup-restore-limit`.
1. Longhorn remove `longhorn.io/backup-restore-priority: in-progress` when the backup is restored.
1. Longhorn determines which engine should start the backup restore by the lowest `longhorn.io/backup-restore-priority` value.

## Design

### Implementation Overview

#### `concurrent-backup-restore-limit` Setting

This setting controls how many volumes restoring from the backup at the same time. Longhorn blocks the backup restore once the restoring engine count exceeds the limit. 

Set the value to **0** to disable backup restore.

```
Category = ettingCategoryDangerZone,
Type     = integer
Default  = 5
```

#### `longhorn.io/backup-restore-priority` Label

1. Tag Longhorn Engine when volume needs to restore from backup. The value is the time in Unix nano. Then let the engine be in the restore back off. This allows Longhorn to tag all Engines when there are concurrent volumes requested for backup restore.
1. Tag  Longhorn Engine when backup restore started. The value is `in-progress`. The number of Longhorn Engines with this labeled value will be compared to the [concurrent-backup-restore-limit](#concurrent-backup-restore-limit-setting) setting value.

### Test plan

- Test the setting should block backup restore when multiple volumes are created from the backup at the same time.

### Upgrade strategy

`None`

## Note [optional]

`None`
