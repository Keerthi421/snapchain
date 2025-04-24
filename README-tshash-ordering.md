# Messages in a Block Should be Ordered by tsHash

## Issue Summary

This change ensures that messages in a block are ordered by their `tsHash` (timestamp + hash) rather than just their timestamp. This ordering is crucial because it guarantees that events cannot rollback account state for clients that perform optimistic updates.

## Problem Description

When messages are not ordered by `tsHash`, we can encounter a scenario where:
- A client submits message m1 at time t1
- Another client submits message m2 at time t2
- If the messages are ordered as m2, m1 in a block, the events emitted will be a MERGE_MESSAGE and a MERGE_FAILURE

This can cause inconsistency for clients, as the first client processing the events may actually undo m2 if it processes the MERGE_FAILURE normally.

## Solution

We modified the `mempool_key()` methods for both user messages and validator messages to ensure that they are ordered by `tsHash` in the mempool. Since the mempool is a `BTreeMap` that orders keys, this ensures that when messages are pulled for a block in `pull_messages()`, they are already in the correct order.

For user messages:
- We now use the entire `tsHash` (timestamp + message hash) as part of the identity in the `MempoolKey`
- This ensures consistent ordering even for messages with the same timestamp

For validator messages:
- We've improved the identity format to ensure stable ordering
- Timestamps are zero-padded to maintain lexicographical ordering

## Testing

We added a test case `test_messages_ordered_by_ts_hash()` that verifies messages with the same timestamp but different hashes are ordered correctly in a `BTreeMap`.

## Benefits

- Ensures that events are processed in a consistent order
- Prevents potential rollbacks of account state for clients doing optimistic updates
- Improves determinism in the system by making message ordering more predictable 