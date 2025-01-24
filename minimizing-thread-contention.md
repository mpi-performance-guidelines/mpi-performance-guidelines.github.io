---
layout: page
title: Minimizing Thread Contention Using Communicator Objects
---

The MPI-2 standard defined thread support levels for MPI so that
multi-threaded programs could use MPI effectively. These support levels
are unchanged in the latest version of the MPI Standard[^1]. They are:

- `MPI_THREAD_SINGLE`
- `MPI_THREAD_FUNNELED`
- `MPI_THREAD_SERIALIZED`
- `MPI_THREAD_MULTIPLE`

From an implementation perspective, the first three levels are not *that* different. Each of the three means that MPI is only being called by a single thread at a time. The thread-safety requirements for such usage are minimal, mainly having to do with dependent libraries or operating system interfaces used by MPI. Such external API calls would not commonly be on the data path, therefore they should not impact communication performance.

`MPI_THREAD_MULTIPLE` is a different story. The MPI specification states that "multiple threads may call MPI, with no restrictions". Thread-safe MPI libraries must identify code paths where shared reources are accessed and add protection to prevent data races or corruption. For communication resources like shared memory or network queues, accesses from multiple threads concurrently results in lock contention and dramatically reduced communication throughput[^2].

For years following MPI-2, `MPI_THREAD_MULTIPLE` was known to have performance issues. Users (and implementors) tended to stick with "MPI everywhere" for the best performance. But as core counts grew, the amount of memory per core could be seen to shrink from generation to generation of machine. Users understandably became tempted by multithreaded optimization to reduce memory pressure on their applications. If those applications attempted to scale out[^3] using MPI concurrently, they would come to know the bottlenecks associated with lock contention.

Research into mitigating the contention caused by concurrent access to shared resources continues, but an alternative approach emerged to expose isolated resources from MPI. MPI could internally maintain multiple communication "contexts" which are functionally independent from one another. By carefully mapping accesses to MPI resources at the application-level, one could exploit the independent contexts and recover much of the "MPI everywhere" performance. Importantly, these application changes could be done without any extensions to the MPI API. The MPI standard already provided a convenient resource isolation mechanism in the form of the MPI communicator object[^4].

<table style="table-layout:fixed">
    <tr>
        <th align="center">MPI Everywhere</th>
        <th align="center">MPI_THREAD_MULTIPLE</th>
        <th align="center">MPI_COMM Per Thread</th>
    </tr>
    <tr>
        <td>
            <img src="/assets/images/mpi-everywhere.png">
        </td>
        <td>
            <img src="/assets/images/mpi-thread-multiple.png">
        </td>
        <td>
            <img src="/assets/images/mpi-comm-per-thread.png">
        </td>
    </tr>
</table>

Multiple MPI implementation support a communicator-per-thread mapping to provide the best `MPI_THREAD_MULTIPLE` performance. Because such a feature can be expensive from a resource perspective, it is often disabled by default and requires explicit settings in the build or environment in order to utilize it. In MPICH[^5] releases since 4.0, up to 64 "VCIs" are supported within a single process with the ch4 device build configuration. Users can specify the actual number of VCIs needed (default=1) at runtime with the `MPIR_CVAR_CH4_NUM_VCIS` environment variable.


```c

```

Intel MPI supports a communicator-per-thread mapping in its [`MPI_THREAD_SPLIT` programming model].

[^1]: [https://www.mpi-forum.org/docs/mpi-4.1/](https://www.mpi-forum.org/docs/mpi-4.1/)
[^2]: [https://dl.acm.org/doi/10.1145/2858788.2688522](https://dl.acm.org/doi/10.1145/2858788.2688522)
[^3]: tbd
[^4]: [https://doi.org/10.1145/3392717.3392773](https://doi.org/10.1145/3392717.3392773)
[`MPI_THREAD_SPLIT` programming model]: https://www.intel.com/content/www/us/en/docs/mpi-library/developer-guide-linux/2021-14/mpi-thread-split-programming-model.html
