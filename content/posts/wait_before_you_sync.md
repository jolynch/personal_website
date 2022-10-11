---
title: "Wait Before You Sync"
date: 2022-05-08T11:40:04-04:00
draft: true
---

I work with distributed databases a lot, and one of the number one performance
issues I see is where distributed databases put
[`fdatasync`](https://linux.die.net/man/2/fdatasync) in the hot path of mutations.

> Please stop putting `fdatasync` in the **hot path**. Call it in the
  background every ~10 seconds or 100MiB written as a *performance* measure,
  but your correctness should be based on checksums and redundancy (replicated
  commit logs and replica snapshots) ... _not_ relying on `fsync`.

For a single machine database in the 1990s, `fdatasync` may have played a
correctness role, but modern distributed databases should _only_ use
`fdatasync` as a way to ensure they don't queue infinite data into page cache
and eventually learn of disk failure.

As a thought exercise, let's consider the following operations against a
drive:

{{< highlight c >}}
(1) fd = open("file.db");
(2) write(fd, "1", 1);
(3) fdatasync(fd)

// Now wait ~5 seconds
(4) read(fd, buf)
{{< /highlight >}}

What can the contents in `buf` be? If you answered, "literally anything" _incuding_
the call hanging, you would be correct! The presence of the
`fdatasync` on line 3 does not guarantee that any subsequent read will
observe the correct data, it just guarantees that the kernel will have
attempted to flush data from page cache to disk and, as of Linux 4.13, if the
Kernel knows that it didn't make it there processes with open file descriptors
should get an EIO.

Remember that even when you call `fdatasync` you have to do something
reasonable with the EIO, which in the case of a distributed database is usually
to drop the corrupt data from view and either re-replicate the affected ranges 
from other nodes or possibly even replacing the faulty node with fresh
hardware.

# Show me the numbers
To prove how disastrous adding `fdatasync` into the hot path of a write
path is, let's use a quick benchmark to evaluate the impact of calling
fsync after various block sizes with constant 4KiB writes.

# Concerns
Database engineers often have concerns when I argue we should be waiting before
we `fdatasync`. Let's go through them.

## But I care about Correctness!

Great, I do too! The way to be correct and durable is to run your transactions
through quorums of replicas e.g. via having multiple replicas running
[paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science)) (or Raft if you
like that flavor of consensus better) to admit mutations into the distributed
commit log. Then, have a background thread on every node that opens the data
files and calls `fdatasync` every ~10-30 seconds (or some size like
`100-1000MiB` whichever comes first) so you give your drives nice constant
amounts of work. Finally, put checksums like xxhash into every block of data
written to the commitlog files and any on disk state you write. If checksums
ever fail or you receive an EIO treat the entire block of data in that file as
corrupt.

> See but you recommended `fdatasync`!

Yes I did, but the recommendation is for performance and not correctness. Your
correctness came from paxos and checksums, _not_ `fdatasync`.

## But what about machine reboots!

There are two kinds of machine reboots, intentional ones and unintentional
ones. By all means on intentional restarts call `fdatasync` on every data file
before rebooting, and you probably even want to `happycache` dump so your
page cache will be preserved across the reboot too!

For unintentional reboots either throw the machine away (practice failure
recovery) or recover the ~10-30 seconds of data from neighbors. The reboot
duration was likely longer than the last background fsync by a lot, so you
have to recover the writes you've missed either way (this is where paxos
log replay comes in handy).

# Is this really a big problem?

Yes. Forcing threads to block wait frequently on `fdatasync` of little bits
of of data at a time caps the performance of your database significantly, and
gets you very little in return.

Drives return garbage, machines fail, kernels panic, processes crash. Failure,
is constant in distributed systems, but `fdatasync` in the hot path doesn't
actually solve any of those, replication and checksums do.

Please, wait before you `fsync`.