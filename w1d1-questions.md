Why doesn't the memtable provide a delete API?
The memtable doesn't provide a delete API because LSM-trees use tombstones for deletions. Instead of actually removing data, we insert a special marker (tombstone) that indicates the key has been deleted. This approach maintains the append-only nature of LSM-trees and ensures proper ordering during compaction.

Does it make sense for the memtable to store all write operations instead of only the latest version of a key? For example, the user puts a->1, a->2, and a->3 into the same memtable.
No, it doesn't make sense. Storing multiple versions of the same key in one memtable wastes memory and provides no benefit since only the latest value matters for reads.

Is it possible to use other data structures as the memtable in LSM? What are the pros/cons of using the skiplist?
Yes, alternatives include B+ trees, red-black trees, or hash tables.
Skiplist pros: Lock-free concurrent reads, probabilistic O(log n), simple implementation, good cache performance
Skiplist cons: Higher memory overhead due to multiple pointers, probabilistic guarantees rather than deterministic 

Why do we need a combination of state and state_lock? Can we only use state.read() and state.write()?
We use both because state is wrapped in Arc<RwLock<>> for the actual data, while state_lock is a separate Mutex for coordinating state transitions (like memtable freezing). This prevents race conditions during complex operations that need exclusive access beyond just read/write.

Why does the order to store and to probe the memtables matter? If a key appears in multiple memtables, which version should you return to the user?
Order matters because newer memtables contain more recent writes. We should probe from newest to oldest and return the first match found, which represents the latest version of the key.

Is the memory layout of the memtable efficient / does it have good data locality? (Think of how Byte is implemented and stored in the skiplist...) What are the possible optimizations to make the memtable more efficient?
The skiplist has poor data locality since keys/values are scattered across heap allocations connected by pointers. Optimizations: arena allocation to group data together, key-value compression, prefix compression for similar keys, or using more cache-friendly structures like B+ trees.


So we are using parking_lot locks in this course. Is its read-write lock a fair lock? What might happen to the readers trying to acquire the lock if there is one writer waiting for existing readers to stop?
No, parking_lot RwLock is not fair by default - it prioritizes performance over fairness. New readers can still acquire the lock even when a writer is waiting, potentially causing writer starvation. This can lead to writers waiting indefinitely if readers keep arriving.


After freezing the memtable, is it possible that some threads still hold the old LSM state and wrote into these immutable memtables? How does your solution prevent it from happening?
Yes, it's possible if threads hold old state references. The solution uses the state_lock mutex to coordinate freezing operations and ensures all write operations acquire fresh state references, preventing writes to frozen memtables.

There are several places that you might first acquire a read lock on state, then drop it and acquire a write lock (these two operations might be in different functions but they happened sequentially due to one function calls the other). How does it differ from directly upgrading the read lock to a write lock? Is it necessary to upgrade instead of acquiring and dropping and what is the cost of doing the upgrade?
Dropping and re-acquiring allows other threads to potentially acquire locks in between, while upgrading maintains exclusive access. However, parking_lot doesn't support lock upgrading, so drop-and-reacquire is necessary. The cost is potential contention and state changes between the operations, but it's simpler than maintaining upgrade-capable locks.