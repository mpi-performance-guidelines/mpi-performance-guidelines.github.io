---
layout: page
title: An Alternative to MPI_ALLTOALLV for Sparse Data Exchange
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

[^1]: [https://doi.org/10.1145/1837853.1693476](https://doi.org/10.1145/1837853.1693476)
