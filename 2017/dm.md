# struct

##`dm_target`

mapping rule

##`dm_table`
mapping table


##`mapped_device`

dm device

##`dm_dev_internal`

block device in mapping rules

##`target_type`

###return of `dm_map_fn`

* `DM_MAPIO_SUBMITTED`: target将提交或者已提交该IO，目标会负责发起请求并向上层报告请求执行的结果，device mapper core无需再关注
* `DM_MAPIO_REMAPPED`: device mapper core应该使用新的设备和扇区范围重新发起请求
* `DM_MAPIO_REQUEUE` or < 0：target希望将I/O退回重新处理 ?
 

###return of `dm_endio_fn`

* `DM_ENDIO_INCOMPLETE`：I/O没有完成
* `DM_ENDIO_REQUEUE` or < 0: 目标希望将I/O退回重新处理？

##`hash_cell`

* find `mapped_device` by uuid or name
* save the inactive `dm_table`

#initialization

##usage

* create: `DM_DEV_CREATE`
* load: `DM_TABLE_LOAD`
* resume: `DM_DEV_SUSPEND`

##create

`mapped_device`

##load

# implementation

## `DMF_NOFLUSH_SUSPENDING`

set on `__dm_suspend` when `noflush` is true.

`noflush` and `do_lockfs` control whether or not to invoke `lock_fs(md)`.

noflush suspending is used to let the target to return the deferred IO to dm
core and resend the IO to the target after resume.

It was introduced by commit 2e93ccc193 [1]:
```
In device-mapper I/O is sometimes queued within targets for later processing.
For example the multipath target can be configured to store I/O when no paths
are available instead of returning it -EIO.

This patch allows the device-mapper core to instruct a target to transfer the
contents of any such in-target queue back into the core.  This frees up the
resources used by the target so the core can replace that target with an
alternative one and then resend the I/O to it.  Without this patch the only
way to change the target in such circumstances involves returning the I/O with
an error back to the filesystem/application.  In the multipath case, this
patch will let us add new paths for existing I/O to try after all the existing
paths have failed.
```

### `dec_pending`

#### supersde any I/O errors

* `io->error > 0 && __noflush_suspending(md)`: keep `io->error`
    * keep the error number before suspending without IO flush
    * keep the first error number during noflush suspending

* `bio_list_add_head(&md->deferred, io->bio)`
    * the `io->error` will be cleared by `__split_and_process_bio`

```
if (error) {
    if (!(io->error > 0 && __noflush_suspending(md)))
        io->error = error;
}
```
