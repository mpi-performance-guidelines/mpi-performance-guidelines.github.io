---
layout: page
title: An Alternative to MPI_ALLTOALLV for Dynamic Sparse Data Exchange
---

For this guide, we are going to lean heavily on the work for Torsten
Hoefler[^1] in designing a scalable communication protocol for dynamic
sparse data exchange using standard MPI-3 functionality. Over the years,
when discussing MPI scalability issues with users, I often point to this
paper as a way to improve their communication performance. The reason is
fairly simple: `MPI_ALLTOALL` is expensive at scale. In fact, it is a
good rule of thumb to assume any of the collective calls with "all" in
the name are expensive, since every process contributes something to the
eventual output seen at every other process.

For users with sparse data, the first step they might take to address
poor communication performance is to look at
`MPI_ALLTOALLV`. `MPI_ALLTOALLV` allows each process to contribute
different amounts of data to the collective, thereby reducing the
overall communication load on the system. This can be effective when the
communication pattern is regular over time, but not so much when it is
irregular. The issue is that `MPI_ALLTOALLV` requires _each process_ to
specify the amount of data contributed by _every other process_ in the
collective. If this information is not known a priori, then typically a
user will first have to perform an `MPI_ALLGATHER` with the necessary
sendcounts of each process before starting the `MPI_ALLTOALLV`. Remember
what we said about collectives with "all" in the name -- they are
expensive. And two collectives is more expensive than one!

What other options does MPI provide? When neighborhood collectives were
originally proposed to the MPI Forum, they were called "sparse"
collectives. However, these operations assume a fixed set of
neighbors. Any changes to the process communication topology requires a
new, expensive to create communicator object with the updated topology
attached. Any potential performance gains from the more sparse
communication operations will likely be wiped out by the overhead of
creating new topology-aware communicators.

MPI one-sided communication is well-suited to irregular communication
patterns, but real-world adoption in applications is low. The reason
being that the one-sided model is more complex to program, both from an
application perspective _and_ an MPI implementation perspective. This
leads to unpredictable performance from system to system that often
rquires deep understanding of the architecture to diagnose and address.

If we are limited to traditional point-to-point messaging and
collectives, but want to avoid using an "all" collective due to
scalability issues, what is left? The key is another collective
operation added in MPI-3 that was initially not thought to have much use
-- `MPI_IBARRIER`. Over the years, the usefulness of blocking
`MPI_BARRIER` outside of benchmarking applications has been called into
question.

[^1]: [https://doi.org/10.1145/1837853.1693476](https://doi.org/10.1145/1837853.1693476)
