# Parallelization

## Floating-point calculations

Historically, floating-point calculations were parallelized in a manner that ensures that the number of workers does not affect the result.
This requires some care in designing algorithms to ensure the exact same result was obtained during floating-point calculations.
The aim was to allow users to freely alter the number of workers without compromising reproducibility.
This is most relevant when dealing with downstream algorithms based on ranks, where slight floating-point errors can lead to discrete changes in rank.

With the benefit of hindsight, this policy is probably unnecessary.

- Most users will not change the number of workers after their analysis,
  so they won't even know that there is a difference due to the number of workers.
- The main reason to change the number of workers is to account for differences in hardware.
  At that point, it is likely that different CPU architectures/instruction sets/standard math libraries are involved,
  at which point exact floating-point reproducibility is no longer guaranteed.
- This policy is inconsistent with the acceptance of different results between matrix representations due to sparsity or row/column preferences.
  The obvious candidate is that of the per-row/column variances, and no one has complained so far. 

All future algorithmic choices in **tatami** can disregard this policy.
This allows us to use more efficient methods for sharing tasks between workers, such as reductions on floating-point values that were computed within each worker.
For example, when computing row sums for a column-major matrix, we could assign blocks of columns to each worker;
compute the partial sum for each row within each worker; and then add the partial sums across workers to obtain the overall row sum.
This would yield slightly different results to worker-agnostic approach where blocks of rows are assigned to each worker,
but would be more efficient as there are fewer stop-starts and cache misses from reading small parts of each column in each worker.

Keep in mind that it's a not a total free-for-all here.
Any differences in results due to parallelization should be limited to floating-point error, i.e., the result should be the same under exact math.
Additionally, the results should still be exactly reproducible within the same compute environment.
This means that the partitioning of tasks and the order of any reductions should be deterministic.
(For comparison, OpenMP has reduction clauses but does not mandate determinism,
which makes it [somewhat difficult to use](https://stackoverflow.com/questions/75211606/openmp-reproducible-reduction).)

## False sharing

False sharing can be mostly avoided by writing code in a manner that uses automatic variables allocated on each thread's stack.
This is good practice regardless of whether we're in a multi-threaded context.

If threads need to write to the heap, false sharing is harder to avoid.
For example, each thread will typically create its own `tatami::Extractor` instance on the heap.
Modifications to data members of each `tatami::Extractor` instance during `fetch()` calls could be another source of false sharing.
This could be fixed by adding an `alignas(hardware_destructive_interference_size)` to the class definition,
but this is only supported by very recent compilers and I'm not even sure if it's legal when the object's natural alignment is stricter.

Fortunately, modern allocators with multi-threaded support will partition the heap into separate "arenas" for each thread.
Allocations in one thread will use memory from one arena, relatively isolated from another thread's allocations in a different arena.
Thus, we can encourage `malloc` to mitigate false sharing by simply creating our heap-allocated objects (e.g., `tatami::Extractor`, `std::vector`) within each thread.
This is convenient as it just involves writing idiomatic C++ where the objects are scoped within each thread. 
Of course, we're not guaranteed to avoid false sharing here - to do that, we'd need to align all heap allocations to `hardware_destructive_interference_size`,
but then we would need to override `std::allocator` in all STL containers, which seems too intrusive for general use.

At some point, we may need to write to contiguous memory in the heap across multiple threads, typically to store our results. 
This should be done sparingly, e.g., by operating solely on thread-local data structures and moving the data to a shared output buffer at the very end.
For example, if we wanted to count the number of non-zero elements in each column of a `tatami::Matrix`, we could do the following:

```cpp
const int NR = matrix->nrow();
const int NC = matrix->ncol();
std::vector<double> column_counts(NC);

tatami::parallelize([&](int, int start, int length) -> void {
    std::vector<double> buffer(NR);
    auto ext = matrix->dense_column();

    // Hopefully this is allocated in a separate arena.
    std::vector<double> resbuffer(length);

    for (int c = start, end = start + length; c < end; ++c) {
        auto ptr = ext->fetch(c, buffer.data());
        int count = 0;
        for (int r = 0; r < NR; ++r) {
            count += ptr[r] != 0;
        }
        resbuffer[c - start] = count;
    }

    // Maybe there's some false sharing here, but we're already out of the
    // loop, so it won't matter much as it's only called once per thread.
    std::copy(resbuffer.begin(), resbuffer.end(), column_counts + start);

    // A slightly more clever implementation could even avoid the allocation
    // and copy in one of the threads by directly writing to 'column_counts'.
    // If only one thread is writing there, there won't be much risk of false
    // sharing, especially given that we're only writing to a slice of
    // 'column_counts' so there's lots of padding until the next allocation.
}, NC, nthreads);
```

With all that said, we shouldn't be too scared of false sharing in typical **tatami** applications.
Most arrays are quite large (> 1000's of elements, typically `int` or `double`) and each thread will only be operating within a small range of that array.
This far exceeds the usual size of a cache line (64 bytes) - so, even in a suboptimal scenario where multiple threads operate within the same array,
any false sharing would limited to the boundaries of each thread's operation and have negligible impact on the overall runtime.

## Optimizing reductions

In some parallel applications, it may be convenient to obtain results from each worker and reduce them in the serial section.
This is easy enough to do with the worker ID that is passed along by `tatami::parallelize()`,
which can index into an output array for storage of each worker's results.
We generally recommend using the following pattern:

```cpp
// Number of workers had better be positive!
std::vector<std::optional<ResultType> > per_worker_output(nworkers);

auto num_used = tatami::parallelize([&](int t, int start, int length) -> void {
    ResultType current = some_kind_of_initialization();
    // Populate 'current' with some results...
    per_worker_output[t] = std::move(current);
}, ntasks, nworkers);

if (num_used == 0) {
    per_worker_output[0] = some_kind_of_initialization();
}

auto base = std::move(*(per_worker_output[0]));
for (int t = 1; t < num_used; ++t) {
    const auto& res = *(per_worker[t - 1]);
    // combine 'res' with '*base' for the reduction.
}
```

In each worker, `current` refers to a thread-specific instance of the result object, which reduces the risk of false sharing.
Once the compute is done, we move the populated result object to the output vector for reduction.
To save time, we only bother traversing the results that were actually populated (`num_used`).
However, if no results were populated, we fill in the first result to make sure that downstream code actually has something to work with.

## SIMD vectorization

Ah, vectorization: the policy with the most flip-flopping in this guide.

Most of the loops in **tatami** are trivially auto-vectorizable so no further work is necessary when compiled with the relevant flags, e.g., `-O3`, `-march=`.
There might be a slight inefficiency due to the need to check against aliasing pointers, but this is fine;
the check is performed outside of the loop and should not have much impact on performance.
On the rare occasion that the check is run frequently relative to the loop body, branch prediction should eliminate most of the cost for the expected case of non-aliasing pointers.
Auto-vectorization also tends to use unaligned SIMD loads and stores, but it's hard to imagine doing otherwise in library code that might take pointers from anywhere (e.g., R).
Fortunately, it seems like unaligned instructions are pretty performant on modern CPUs,
see commentary [here](https://stackoverflow.com/questions/52147378/choice-between-aligned-vs-unaligned-x86-simd-instructions)
and [here](https://stackoverflow.com/questions/20259694/is-the-sse-unaligned-load-intrinsic-any-slower-than-the-aligned-load-intrinsic-o).

That said, there are two common scenarios where auto-vectorization is not possible.

1. Accessing a sparse vector by index.
   If the compiler cannot prove that the indices are unique, different loop iterations are not guaranteed to be independent. 
   Here, I went through many possibilities to inform the compiler that vectorization can be performed.
   
   - Using OpenMP SIMD to force vectorization. 
     While this works, it is often subject to a more heavy-handed interpretation by compilers.
     Upon seeing `#pragma omp simd`, [GCC](https://developers.redhat.com/articles/2023/12/08/vectorization-optimization-gcc) will forcibly vectorize the loop,
     even if doing so would decrease performance according to its cost model.
     [MSVC](https://devblogs.microsoft.com/cppblog/simd-extension-to-c-openmp-in-visual-studio/) goes further and enables fast floating-point inside the loop, which is not generally desirable.
   - Encouraging vectorization with compiler-specific pragmas to indicate that there are no dependencies between loop iterations,
     e.g., `#pragma GCC ivdep` or `#pragma clang loop vectorize(assume_safety)`.
     This also works fairly well but it's hard to keep track of exactly how each compiler is interpreting these pragmas.
     Are all dependencies ignored? Or just the non-proven ones?
     Needless to say, these pragmas are non-portable and require careful testing on each platform.

   In the end, I suspect that it's not worth the effort to attempt to optimize this access pattern.
   Plenty of excuses here: lack of efficient gather/scatter for many CPUs, likely memory bottlenecks in sparse data, etc.
2. Calling `<cmath>` functions inside the loop, primarily for the delayed math operations.
   Vectorization is precluded by the `errno` side-effect and, for some functions like `std::log`, the need for `-ffast-math` to use vectorized replacements.
   While compiling with `-fno-math-errno` seems [fine](https://stackoverflow.com/questions/7420665/what-does-gccs-ffast-math-actually-do?noredirect=1&lq=1),
   using fast math in general is obviously a less palatable option and I can't cajole GCC into using the (unexported) vectorized `log` function without it.
   So, the solution is to just write a variant of a delayed operation helper that uses a vectorized `log` implementation from an external library like **Eigen**.
   Then, users can choose between the standard helper or the vectorized variant that requires an extra dependency.
