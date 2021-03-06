
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

* `thin_c`
    * `dm_target->private = tc`
        * `dm_thin_device`
            * `thin_c->td = td`

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

### `thin_bio_map`

* bio prision
    * hold ?
    * virtual cell
        * within the same block of a thin device
    * data cell or physical cell
        * within the same block of the data device

* bio hook ?

* `__find_block`
    * `dm_btree_lookup(pmd->nb_info, pmd->root, {id, block}, &value)`
    * `unpack_lookup_result`
        * result->shared = __snapshotted_since(td, blk_time)
            * if the block is provisioned before snapshot, it's shared

* defer
    * bio:
        * flush/discard
        * bio in virtual cell
            * when the physical is found and it's not shared
        * bio in physical cell
            * when the physical cell is appending for process
    * cell:
        * virtual cell when the status of its physical block is shared or unknown

* throttle
    * flush and discard bio

* prefetch ?

* prepare ?
    * mapping
        * `struct dm_thin_new_mapping`: throttle the creation of new mapping ?
    * discard
    * `discard_pt2`

* `process_deferred_bios`
    * for each thin process cells and bios
    * commit if timeouted or got flush bios

* `provision_block`
    * `alloc_data_block`
        * `dm_pool_alloc_data_block`
            * `dm_sm_new_block(pmd->data_sm)`
                * find a block in bitmap btree with a ref cnt valued as 0
                * inc the ref cnt of the free block
    * `schedule_zero`
        * `process_prepared_mapping((struct dm_thin_new_mapping *)m)`
            * `dm_thin_insert_block`
                * `dm_btree_insert_notify`
            * remap bio or cell

* quiesce during copy: for block sharing
    * add `dm_thin_new_mapping` to `pool->shared_read_ds`
    * `dm_deferred_entry_inc` when read the shared block
        * `h->shared_read_entry`
    * `thin_endio`

* `inc_all_io_entry`
    * `h->all_io_entry` for discard ?
        * `thin_endio`

* `process_shared_bio`
    * `break_sharing`
        * alloc_data_block
        * schedule_internal_copy
            * schedule_copy
                * dm_kcopyd_copy
                * complete_mapping_preparation
                    * atomic_dec_and_test(&m->prepare_actions)
                    * defer to worker: add to `pool->prepared_mappings`
                        * remap and issue directly ?

    * `cell_defer_no_holder`


* sequence
    * issue sequence ?
    * out-of-order ?
        * fs will wait for the completion of bio_1 ?
        * bio_1, bio_2 + flush + fua
            * bio_1 is deferred in a cell
            * a flush without data from dm
                * flush is defered to defered bio
            * bio_1 needs break sharing and is deferred again
            * flush ??

## b-tree layout

* `dm_btree_info`

* `btree_node`: leaf node and internal node

* `shadow_spine`
    * current btree node and its parent node

* `ro_spine`
    * a readonly variant of `shadow_spine`

### thin detail b-tree

* `pmd->detail_info`
    * 1 level
    * no inc/dec/equal

### data block mapping b-tree

* `pmd->info`
    * context: `pmd->data_sm`
    * inc/dec/equal: `data_block_inc|dec|equal`
    * 2 level
        * it can be used a 1-level b-tree
            * thin id to a sub b-tree
                * `pmd->tl_info`
                * scenario: new thin and delete thin
                    * new a b-tree (bl_info), and insert it
            * virtual block to block_time b-tree
                * `pmd->bl_info`
                * scenario: remove block range of a thin device

