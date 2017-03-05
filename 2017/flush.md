
=`block/blk-flush.c`

==essence

* double buffer for flush request list
* only one flush is in progress
* defer the flush if any request is executing DATA
* twice completion for flush request with data by using `RQF_FLUSH_SEQ`
* add PREFLUSH and POSTFLUSH req to the tail of the `queue_head`
* add DATA req to the head of the `queue_head`
* when flush is inflight, `__elv_next_request` will return NULL for flush-req unqueueable queue

==`blk_insert_flush`

=examples

==one flush

write cached enabled, but without FUA support: policy is PREFLUSH | DATA | POSTFLUSH

```c
blk_insert_flush
    * check rq
        * rq->bio == rq->biotail
    * update rq
        * rq->rq_flags |= RQF_FLUSH_SEQ
        * rq->flush.saved_end_io = rq->end_io
        * rq->end_io = flush_data_end_io
    * start flush machinery for PREFLUSH
        * rq->flush.seq |= 0
        * fq->flush_pending_idx = 0
        * fq->flush_running_idx = 0
        * fq->flush_pending_since = jiffies
        * list_move_tail(&rq->flush.list, &fq->flush_queue[0])
        * toggle pending_idx: fq->flush_pending_idx = 1 (flush is in flight)
        * initialize fq->flush_rq
            * blk_rq_init
            * rq->cmd_flags = REQ_OP_FLUSH | REQ_PREFLUSH
            * rq->rq_flags |= RQF_FLUSH_SEQ
            * rq->rq_disk = first_rq->rq_disk
            * rq->end_io = flush_end_io
        * insert to the tail of the queue
            * list_add_tail(&rq->queuelist, &rq->q->queue_head)
        * blk_run_queue
            * blk_peek_request
                * __elv_next_request
                    * fetch from check q->queue_head
                    *  flush_pending_idx == flush_running_idx && !queue_flush_queueable(q)
                        * fq->flush_queue_delayed = 1
                        * return NULL
        * flush_end_io
            * flush_running_idx = 0
            * running = &fq->flush_queue[0]
            * toggle running_idx: the completion of the flush request
                * flush_running_idx = 1
            * elv_completed_request(q, flush_rq)
            * iter rq in running list
                * advance flush machinery: PREFLUSH
                    * rq->flush.seq |= PREFLUSH
                    * list_move_tail(&rq->flush.list, &fq->flush_data_in_flight) * list_add(&rq->queuelist, &rq->q->queue_head)
            * check fq->flush_queue_delayed
            * kick queue: blk_run_queue_async
            * clear fq->flush_queue_delayed
        * flush_data_end_io
            * elv_completed_request
            * rq->rq_flags &= ~RQF_STARTED;
            * advance flush machinery: DATA
                * rq->flush.seq |= DATA
                * flush_pending_idx = 1
                * flush_running_idx = 1
                * list_move_tail(&rq->flush.list, &fq->flush_queue[1])
                * toggle pending_idx: flush_pending_idx = 0
                * initialize fq->flush_rq
                * insert to the tail of the queue
                    * list_add_tail(&rq->queuelist, &rq->q->queue_head)
        * flush_end_io
            * flush_pending_idx = 0
            * flush_running_idx = 1
            * running = fq->flush_queue[fq->flush_running_idx]
            * toggle running idx: fq->flush_running_idx = 0
            * elv_completed_request
            * for each rq in running list
                * advance flush machinery: POSTFLUSH
                    * rq->flush.seq |= POSTFLUSH
                    * DONE
                        * list_del_init(&rq->flush.list)
                        * restore rq
                        * rq->bio = rq->biotail
                        * rq->rq_flags &= ~RQF_FLUSH_SEQ
                        * rq->end_io = rq->flush.saved_end_io
                    * __blk_end_request_all
            * kick the queue to avoid stall

```
==two flush

write cache enabled, and without FUA support.

* the first flush A: PREFLUSH | DATA
* the second flush B: PREFLUSH | DATA

```c
* policy = PREFLUSH_A | DATA_A
* update and save the original end_io
* start flush machinery for A: 0
    * add rq_a->flush to the tail of flush_queue[0]
    * get pending: flush_queue[0]
    * toggle pending_idx to 1
    * add fq->flush_rq to the tail of q->queue_head

* policy = PREFLUSH_B | DATA_B
* start flush machinery for B: 0
    * add rq_b->flush to the tail of flush_queue[1]

* PREFLUSH_A complete: flush_end_io
    * get running: flush_queue[0]
    * toggle running_idx to 1
    * start flush machinery for A: PREFLUSH
        * rq_a->flush to flush_data_in_flight
        * add rq_a to the head of q->queue_head
        * get pending: flush_queue[1]
        * the pending timeouts
            * toggle pending_idx to 0
            * add rq_b->flush to the tail of q->queue_head

    * DATA_A complete: flush_data_end_io (queue head)
        * start flush machinery for A: DATA
            * the flush machinery has done

    * PREFLUSH_B complete: flush_end_io (queue tail)
        * get running: flush_queue[1]
        * toggle running_idx: running_idx = 0
        * start flush machinery for B: PREFLUSH
```

=question

==filesystem consistency

write cache: `request_queue->queue_flags & (1 << QUEUE_FLAG_WC)`

fua: `queue_flags & (1 << QUEUE_FLAG_FUA)`

* no write cache
    * `REQ_PREFLUSH` and `REQ_FUA` will be clear in function `generic_make_request_checks`:
        ```c
        if (op_is_flush(bio->bi_opf) && !test_bit(QUEUE_FLAG_WC, &q->queue_flags)) {
            bio->bi_opf &= ~(REQ_PREFLUSH | REQ_FUA)
        }
        ```

