
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

## create snapshot

