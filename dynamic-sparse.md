---
layout: page
title: An Alternative to MPI_ALLTOALLV for Dynamic Sparse Data Exchange
---

For this guide, we are going to lean heavily on the work for Torsten
Hoefler[^1] in designing a scalable communication protocol for dynamic
sparse data exchange using standard MPI-3 functionality. Over the years,
when discussing MPI scalability issues with users, I often point to this
paper as a potential way to improve their application. The reason is
fairly simple: `MPI_ALLTOALL` is expensive at scale. In fact, it is a
good rule of thumb to assume any of the collective calls with "all" in
the name are expensive, since every process contributes something to the
eventual output seen at every other process. These types of collectives
in practice act as a global synchronization.

For users with sparse data, the first step they might take to address
poor communication performance is to look at
`MPI_ALLTOALLV`. `MPI_ALLTOALLV` allows each process to contribute
different amounts of data to the collective, thereby reducing the
overall communication load on the system. This can be effective when the
communication pattern is regular over time, but not so much when it is
irregular. The issue is that `MPI_ALLTOALLV` requires _each process_ to
specify the amount of data contributed by _every other process_ in the
collective. If this information is not known _a priori_, then typically
a user will first have to perform _another_ `MPI_ALLTOALL` with the
necessary sendcounts for each process before starting the
`MPI_ALLTOALLV`. Remember that we are trying to avoid "all"
collectives. Performing a second one is not likely to help performance!

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
requires deep understanding of the architecture to diagnose and address.

If we limit ourselves to traditional point-to-point messaging and
collectives, but want to avoid using an "all" collective due to
scalability issues, what is left? The key is another collective
operation added in MPI-3 -- `MPI_IBARRIER`. It is may not be immediately
obvious how a nonblocking barrier is useful to applications, but the key
is in the way that participants _notify_ that they have reached a
certain point in the execution, but do not block on that
synchronization. Rather, they are free to continue communicating with
other processes that have not yet reached the synchronization. The use
of `MPI_IBARRIER` and nonblocking synchronous send operations allows
each process to dynamically modify its communication neighbors in a more
lightweight manner than a traditional collective approach. The
literature refers to this as the "nonblocking census" algorithm.

### Example

Let's take a look at an [example][ex] using both `MPI_ALLTOALLV` and the
nonblocking census algorithm to see how they perform in practice. To
simulate the dynamic nature the communication, we will randomly select
communication partners. Each process communicates with a "sparse" 10% of
other processes. The size of each messages is constant for simplicity.

#### `MPI_ALLTOALLV` data exchange
```c
for (int i = 0; i < NUM_ITERS; i++) {
    int num_neighbors;
    int *neighbors = get_neighbors(&num_neighbors);

    start = MPI_Wtime();
    memset(sendcounts, 0, sizeof(int) * wsize);
    for (int j = 0; j < num_neighbors; j++) {
        sendcounts[neighbors[j]] = NUM_DOUBLES;
    }
    MPI_Alltoall(sendcounts, 1, MPI_INT, recvcounts, 1, MPI_INT, MPI_COMM_WORLD);
    MPI_Alltoallv(sendbuf, sendcounts, displs, MPI_DOUBLE, recvbuf, recvcounts,
                  displs, MPI_DOUBLE, MPI_COMM_WORLD);
    end = MPI_Wtime();
    total_time += end - start;
}
```

MPI collectives are attractive for their brevity. A lot of complexity is
abstracted away by the collective interfaces, resulting in relatively
short code. Compare that to the nonblocking census next.

#### Nonblocking Census (`MPI_IBARRIER` + `MPI_ISSEND`)
```c
for (int i = 0; i < NUM_ITERS; i++) {
    /* determine neighbors for this iteration */
    int num_neighbors;
    int *neighbors = get_neighbors(&num_neighbors);

    start = MPI_Wtime();
    for (int j = 0; j < num_neighbors; j++) {
        int dest = neighbors[j];
        double *buf = sendbuf + (dest * NUM_DOUBLES);

        MPI_Issend(buf, 16, MPI_DOUBLE, dest, 0, MPI_COMM_WORLD, &sreqs[j]);
    }

    MPI_Request barrier_req = MPI_REQUEST_NULL;
    while (1) {
        MPI_Status status;
        int flag;

        /* check for incoming messages */
        MPI_Iprobe(MPI_ANY_SOURCE, 0, MPI_COMM_WORLD, &flag, &status);
        if (flag) {
            int src = status.MPI_SOURCE;
            double *buf = recvbuf + (src * NUM_DOUBLES);

            MPI_Recv(buf, NUM_DOUBLES, MPI_DOUBLE, src, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        }

        if (barrier_req != MPI_REQUEST_NULL) {
            MPI_Test(&barrier_req, &flag, MPI_STATUS_IGNORE);
            if (flag) {
                /* barrier completed, this iteration is done */
                free(neighbors);
                end = MPI_Wtime();
                total_time += end - start;
                break;
            }
        } else {
            MPI_Testall(num_neighbors, sreqs, &flag, MPI_STATUSES_IGNORE);
            if (flag) {
                /* sends are done, start barrier */
                MPI_Ibarrier(MPI_COMM_WORLD, &barrier_req);
            }
        }
    }
}
```

As you can see, this version is more lengthy. As a user, there must be a
clear benefit to the longer version if it is to be developed and
maintained in application codes. For our experiment, we will run the
example for 1000 iterations of each algorithm and compare the average
communication time. For this run we use 8 JLSE Skylake nodes with 56
processes per node to fully subscribe the CPU cores. The MPI in use is
MPICH 4.2.3.

#### Experiment

```console
# mpiexec -bind-to core -ppn 56 ./dsde
avg alltoallv time = 27.182003
avg nonblocking census time = 18.780934
```

### Takeaway

The result about shows about a 30% reduction in communication time using
the nonblocking census algorithm. But this is only for our synthetic
example. Users should consider this approach for their application if it
fits the criteria we discussed. That is, if each process has relatively
few communication partners, and those partners change from iteration to
iteration. If so, it may be worth trying out the alternative and running
some representative experiments for comparison. If the benefits are
significant, they could justify the added complexity in your code!

[^1]: [https://doi.org/10.1145/1837853.1693476](https://doi.org/10.1145/1837853.1693476)
[ex]: https://github.com/raffenet/bssw-examples/tree/main/dsde
