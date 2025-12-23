# Parallelization

## Floating-point calculations

Historically, floating-point calculations were parallelized in a manner that ensures that the number of workers does not affect the result.
This requires some care in designing algorithms to ensure the exact same result was obtained during floating-point calculations.
The aim was to allow users to freely alter the number of threads without compromising reproducibility.
This is most relevant when dealing with downstream algorithms based on ranks, where slight floating-point errors can lead to discrete changes in rank.

With the benefit of hindsight, this policy is probably unnecessary.

- Most users will not change the number of threads after their analysis,
  so they won't even know that there is a difference due to the number of threads.
- The main reason to change the number of threads is to account for differences in hardware.
  At that point, it is likely that a different CPU is involved, at which point exact floating-point reproducibility is no longer guaranteed.
- This policy is inconsistent with the acceptance of different results between matrix representations due to sparsity or row/column preferences.
  The obvious candidate is that of the per-row/column variances, and no one has complained so far. 

All future algorithmic choices in **tatami** can disregard this policy.
This allows us to use more efficient methods for partitioning work between cores. 
Note that the differences in results due to parallelization should be limited to floating-point error, i.e., the result should be the same under exact math.

## False sharing

False sharing can be mostly avoided by writing code in a manner that uses automatic variables allocated on each thread's stack.
This is good practice regardless of whether we're in a multi-threaded context.

If threads need to write to the heap, false sharing can be mitigated by creating separate heap allocations within each thread, e.g., by constructing a thread-specific `std::vector`.
This gives `malloc` a chance to use different memory arenas (depending on the implementation) that separates each thread's allocations.
We could guarantee protection against false sharing by aligning all heap allocations to `hardware_destructive_interference_size`;
but then we would need to override `std::allocator` in all STL containers, which seems too intrusive for general use.
Obviously, writing to contiguous memory in the heap across multiple threads should be done sparingly, typically to the output buffer once all calculations are complete.

An interesting sidenote is that each thread will typically create its own `tatami::Extractor` instance on the heap.
Modifications to data members of each `tatami::Extractor` instance during `fetch()` calls could be another source of false sharing.
This might be avoidable by adding an `alignas(hardware_destructive_interference_size)` to the class definition,
but this is only supported by very recent compilers and I'm not even sure if it's legal when the object's natural alignment is stricter;
so, we'll just stick to our ostrich strategy of hoping that `malloc` takes care of it.

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
