
## time

* `pmd::time`: increase when creating a snapshot device
* `thin_device::creation_time`: the value of `pmd::time` when creating a thin device
* `thin_device::snapshotted_time`
    * the value of `pmd::time` after the creation of snapshot
    * both of the origin device and the snapshot device should be updated
* the time of block
    * the value of `pmd::time`

## b-tree

* info: thin id => meta block nr/ thin blk nr => data block nr + time
* `detail_info`: thin id => `disk_device_details`
* `tl_info`: thin id => 8B ?
    >* Just the top level for deleting whole devices
* `bl_info`: thin id => 8B ?
	>* Just the bottom level for creating new devices.

## space map

### `dm_tm_new_block`

* `dm_sm_new_block`
    * new block X
    * inc the ref count of X
    * clear as zero
    * insert as shadow block

## pool resume

* extent data device size: `dm_pool_resize_data_dev`

* `dm_pool_resize_data_dev`
    * `sm_ll_extend`
        * new data bitmap block
        * insert `disk_index_entry` to data bitmap btree

* `commit`
    * `dm_pool_commit_metadata`
        * `__write_changed_details`: update the details of thin devices
        * `dm_sm_commit(pmd->data_sm)`:
            * write nothing, but save smd->ll, clear smd->begin
            * the root of bitmap btree and ref cnt btree are saved at superblock
        * `dm_tm_pre_commit`
            * `dm_sm_commit(pmd->tm->sm)`
                * metadata_ll_commit
                    * shadow bitmap root
            * `dm_bm_flush(tm->bm)`
                * `__write_dirty_buffers_async`
                * `__flush_write_list`: plug and unplug to merge write
                * wait for io done
                * `dm_bufio_issue_flush`
            * super block
                * save_sm_roots
                * superblock_lock
                * copy_sm_roots
                * dm_tm_commit(pmd->tm, sblock)
                    * wipe_shadow_table
                    * dm_bm_flush
## create thin

* `dm_pool_metadata`
    * `pool_c`: `dm_target::private`
    * `pool`: `pool_c::pool`
    * `dm_pool_metadata`: `pool:pmd`

* `pool_message`
    * `dm_pool_create_thin`
        * new metadata block formatted by `pmd->bl_info`
        * insert the block into `pmd->root` according to `pmd->tl_info`
        * __open_device
            * new `dm_thin_device`
            * add to `pmd->thin_devices`

### dmsetup message

```bash
dmsetup message pool 0 'create_thin 1'
```

### dmsetup create

```bash
dmsetup create thin_1 --table '0 N thin /dev/mapper/pool 1'
```

#### `thin_ctr`

`thin_c`
`dm_thin_device`

* `tc::origin_dev`: external snapshot

* find pool by device name
    * `dm_get_device`: get `dm_dev` of pool
    * `dm_get_md(pool_dev->bdev->bd_dev)`: get `mapped_device` of pool
    * `__pool_table_lookup(pool_md)`: get `pool` from `dm_thin_pool_table.pools`

* `dm_pool_open_thin_device`
    * find in `pmd::thin_devices`

* initialize `dm_target`
    * `max_io_len`
    * `per_io_data_size`: `sizeof(struct dm_thin_endio_hook)`

#### `thin_preresume`

```c
tc->origin_size = get_dev_size(tc->origin_dev->bdev)
```

## create snapshot

### `dm_pool_create_snap`

* `__create_snap`
    * find the mapping tree of the origin
    * inc the ref count of the block
    * insert the block into `pmd->root` according to `pmd->tl_info`
    * `__open_device`
    * `__set_snapshot_details`: the origin device has been updated
        * td->changed = 1
    * `__close_device`

## new block

