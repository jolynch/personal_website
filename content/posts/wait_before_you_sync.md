---
title: "Wait Before You Sync"
date: 2023-07-02T11:40:04-04:00
tags: ["software-opinions"]
---

I work with distributed databases, and one of the number one performance issues
I see is when they put
[`fdatasync`](https://linux.die.net/man/2/fdatasync) calls in the critical path
of accepting writes.

> **Please stop putting `fdatasync` in the hot path**. Call it in the
  background every ~10-30 seconds or O(100MiB) written, as a *performance*
  measure, but your correctness should be based on checksums and redundancy
  — e.g. replicated commit logs and replica snapshots — _not_ relying on
  `fdatasync`.

For a single machine database in the 1990s with only local drives for
durability, `fdatasync` may have played a correctness role, but modern
distributed databases should _only_ use `fdatasync` as a way to ensure they
don't queue unbounded data into page cache, [which would eventually force
synchronous writeback](https://github.com/firmianay/Life-long-Learner/blob/master/linux-kernel-development/chapter-16.md).
Instead, most still-correct-but-higher-performance storage engines prefer to
trigger syncs periodically and asynchronously (not in the critical path of
writes) to avoid writeback, and to eventually learn of disk failure.

Before the SQL database folks and authors of academic distributed systems
papers get out their pitchforks, consider the following pseudo file
operations occurring on an SSD:

{{< highlight c >}}
// First, write some data to a file
(1) fd = open("file.db");
(2) write(fd, "1", 1);
(3) assert(fdatasync(fd) == 0)
(4) assert(close(fd) == 0)

// Then, try to read later
(5) fd = open("file.db")
(6) read(fd, buf)
{{< /highlight >}}

What can the contents in `buf` be? If you answered any of the following you
would be correct:

* `1` - Almost always we get what we wrote
* Nothing - Sometimes drives lose data
* Random corruption - Sometimes drives return garbage
* Can't tell - Sometimes the `read` syscall hangs forever

The presence of the `fdatasync` on line 3 does not guarantee that any
subsequent read (line 6) will observe the correct data. It just guarantees that
the kernel will have attempted to flush data from page cache to disk and --- as of
Linux 4.13 --- if the kernel knows that it didn't make it there, processes with
open file descriptors referencing those pages should get an `EIO`.

Remember that even when you call `fdatasync` you have to do something
reasonable with the `EIO`. In the case of a distributed database, that is
usually to drop the corrupt data from view and either re-replicate the affected
ranges from other nodes or possibly even replace the faulty node with fresh
hardware. If you are running an SQL setup, the equivalent would be to trigger
a failover to a hot standby after taking downtime to play/resolve the remaining
replication stream. *These actions can equally be done on a checksum failure*.

So why call `fdatasync` at all? We still want to sync in the background
because we both want to 1) make sure we are not accepting data faster than our
underlying hardware can actually handle it, and 2) find out
about disk failures in a reasonable amount of time so we can trigger recovery,
ideally while data still lives in a time based replicated commit log or
replication data structure.

# Show me the numbers
To show how disastrous adding `fdatasync` into the hot path of a writes are,
a quick [benchmark](https://github.com/jolynch/performance-analysis/blob/master/notebooks/fsync/benchmark.c)
can be used [to evaluate the impact](https://github.com/jolynch/performance-analysis/blob/master/notebooks/fsync/fsync_after.ipynb)
of calling `fdatasync` after various block sizes with `1GiB` worth of `4KiB`
writes. Since most databases write data in blocks of `4KiB+` (and most physical
drives have block sizes of `4KiB` anyways), this is somewhat of a worst case
test. The benchmark writes `1GiB` of data with different strategies of calling
`fdatasync` with an `ext4` filesystem backed by a single `Samsung SSD 970 PRO
512GB` NVMe flash drive.  This drive theoretically can achieve `~2.7GiB/s`
although in this case it maxes out around ~`1.5GiB/s`.

The time to write `1GiB` with no `fdatasync`, one `fdatasync` at the end of the
`1GiB`, or one call every `100MiB`, `10MiB`, and `1MiB` are as follows:

[![fsync_qualitative](/img/fsync_qualitative.svg)](/img/fsync_qualitative.svg)

It is clear that calling `fdatasync` any more than every ~10MiB starts
to tank performance. To see just how bad it can get, we can run the benchmark
going even further, calling `fdatasync`
[more frequently](https://gist.github.com/jolynch/a67a2bbd235dcbc3a6e1b0d47ea6a3be#file-benchmark-run-sh)
over multiple trials (condolences to my SSD):
{{< highlight text >}}
Strategy  | p50 time-to-write |
-------------------------------
never     |           151.0ms |
end       |           663.5ms |
100MiB    |           738.5ms |
 10MiB    |          1127.0ms |
  1MiB    |          3159.5ms |
512KiB    |          5955.0ms |
256KiB    |         10695.5ms |
128KiB    |         17842.5ms |
 64KiB    |         26912.5ms |
 32KiB    |         52306.0ms |
 16KiB    |        111981.0ms |
{{< /highlight >}}


A plot of this data going out all the way to 16KiB writebacks shows that the
pattern of writes getting unbearably slow as flushes get smaller:
[![fsync_qualitative](/img/fsync_quantitative.svg)](/img/fsync_quantitative.svg)

From this data, it is clear that calling `fdatasync` too often completely tanks
performance on modern fast NVMe drives. I've seen some databases that call
`fdatasync` after _every_ write transaction, typically because of a
misunderstanding of the failure modes of actual hardware.

# Concerns
Database engineers often have concerns when I propose we should be waiting
before we `fdatasync`. Let's go through them.

## But I care about Correctness and Durability!

Great, I do too! The way to be correct and durable in a distributed database is
[via replication](https://vitess.io/docs/18.0/overview/scalability-philosophy/#durability-through-replication):
run your transactions through quorums of replicas either with a `leader ->
follower + witness` setup (~semi-sync replication) or have multiple
replicas running
[Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science)) (or
[Raft](https://raft.github.io/raft.pdf) or [Atomic
broadcast](https://en.wikipedia.org/wiki/Atomic_broadcast) if you like those
flavors better) to admit mutations into the distributed commit log. On each
machine accepting writes to the commit log, write files to ~32-128MiB
segments append-only and with locks provided by the process. Then, have a
background thread that opens the segment files and calls `fdatasync` every
~10-30 seconds (or some size like `100-1000MiB` whichever comes first) so you
give your drives nice constant amounts of work. Remember to use the
[`kyber`](https://www.kernel.org/doc/html/latest/block/kyber-iosched.html) IO
scheduler so your read latencies are not affected by these background flushes.

Finally, put
[_fast_ checksums]({{< relref use_fast_data_algorithms >}}) such as a
[`xxhash`](https://github.com/Cyan4973/xxHash) along with every block of data
written to the commitlog files and any on-disk state you write. When reading
files, the read path must check the checksums for any blocks of data it reads,
and if checksums ever mismatch or result an `EIO`, treat the entire block of
data in that file as corrupt and initiate recovery from replicas or
backups/snapshots.

## But what about machine reboots?

There are two kinds of machine reboots: intentional ones and unintentional
ones. By all means, when rebooting intentionally, call `fdatasync` on every data
file. You probably even want to
[`happycache`](https://github.com/hashbrowncipher/happycache) dump so your
page cache will be preserved across the reboot too!

For unintentional reboots, treat it as a machine failure: either throw the
machine away (practice failure recovery) or recover the ~10-30 seconds of data
from neighbors. The reboot duration, when the node was missing writes, was
likely significantly longer than the last background `fdatasync`, so you will
have to recover the writes you've missed either way; this is where having a
distributed replication log to replay comes in handy.


## But what if I need to see the write in another process?

On Linux with a POSIX compliant filesystem, any
[`write`](https://man7.org/linux/man-pages/man2/write.2.html) syscall has a
strong read-after-write guarantee without an `fdatasync`. If the kernel caches a
page it must maintain that property.

> POSIX requires that a read(2) that can be proved to occur after a
  write() has returned will return the new data.  Note that not all
  filesystems are POSIX conforming.

On most database servers with `xfs` or `ext4` filesystems, `read` syscalls
really should observe `write`s. If the database is bypassing the kernel
entirely and trying to do their own device IO without going through
`read`/`write` syscalls, then naturally this whole post does not apply.

# Is this really a big problem?

**Yes**. Forcing threads to frequently `fdatasync` small bits
of data will cap the performance of your database significantly, and
gets you very little in return. Databases that `fdatasync` after every
transaction or 4KiB tank performance based on a flawed understanding of how
drives (and especially SSDs) actually function.

Drives return garbage, machines fail, kernels panic, processes crash. Failure
is constant in distributed systems, but `fdatasync` in the hot path doesn't
actually solve any of those -- replication and checksums do.

Please, wait before you `fdatasync`.
