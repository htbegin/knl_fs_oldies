=struct

==`dm_target`

mapping rule

==`dm_table`
mapping table


==`mapped_device`

dm device

==`dm_dev_internal`

block device in mapping rules

==`target_type`

===return of `dm_map_fn`

* `DM_MAPIO_SUBMITTED`: target将提交或者已提交该IO，目标会负责发起请求并向上层报告请求执行的结果，device mapper core无需再关注
* `DM_MAPIO_REMAPPED`: device mapper core应该使用新的设备和扇区范围重新发起请求
* `DM_MAPIO_REQUEUE` or < 0：target希望将I/O退回重新处理 ?
 

===return of `dm_endio_fn`

* `DM_ENDIO_INCOMPLETE`：I/O没有完成
* `DM_ENDIO_REQUEUE` or < 0: 目标希望将I/O退回重新处理？

==`hash_cell`

* find `mapped_device` by uuid or name
* save the inactive `dm_table`

=initialization

==usage

* create: `DM_DEV_CREATE`
* load: `DM_TABLE_LOAD`
* resume: `DM_DEV_SUSPEND`

==create

`mapped_device`

==load

