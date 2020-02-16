# AutoPreallocation

[![Build Status](https://travis-ci.com/oxinabox/AutoPreallocation.jl.svg?branch=master)](https://travis-ci.com/oxinabox/AutoPreallocation.jl)
[![Coveralls](https://coveralls.io/repos/github/oxinabox/AutoPreallocation.jl/badge.svg?branch=master)](https://coveralls.io/github/oxinabox/AutoPreallocation.jl?branch=master)


**Have you ever wanted your code allocate less?**
**Have you ever felt explictly preallocating everything was just too hard?**
**Have you ever thought: _"Everytime I run this function the allocations should always be the same"_?**
**Well look no further friend. Its AutoPreallocation.jl**

## How to use:
The process to use AutoPreallocation.jl is two step:
1. Generate a *record* of all allocations
2. Use that *record* to avoid allocations when the function is called

## Example:
```julia
julia> using AutoPreallocation, BenchmarkTools

julia> foo() = ones(1, 2096) * ones(2096, 1024) * ones(1024,1)
foo (generic function with 1 method)

julia> @btime foo()
  2.174 ms (7 allocations: 16.41 MiB)
1×1 Array{Float64,2}:
 2.146304e6

julia> const foo_res, foo_record = record_alloctions(foo)
(value = [2.146304e6], allocation_record = AutoPreallocation.AllocationRecord(
    [Array{Float64,2}(undef, (1, 2096)), Array{Float64,2}(undef, (2096, 1024)), Array{Float64,2}(undef, (1024, 1)), Array{Float64,2}(undef, (1, 1024)), Array{Float64,2}(undef, (1, 1))],
    [(1, 2096), (2096, 1024), (1024, 1), (1, 1024), (1, 1)]
))

julia> @btime avoid_alloctions($foo_record, foo)
  1.376 ms (29 allocations: 672 bytes)
1×1 Array{Float64,2}:
 2.146304e6
```

#### Tip:
To avoid having to rerun `record_alloctions` every time the program is used, you can use `repr` on the record object, and then put it in a `const` in your program.

## Limitations (Important)
Despite the hip introduction, AutoPreallocation is not a tool to use lightly.
It requires you understand the following limitations, which while relatively rare in practice certainly do occur.

AutoPreallocation is also not hugely mature (yet), and if you violate these limitations, it may silently give you the wrong answer.

### Function must always allocate the same way. (per _record_ object)

You cannot reuse the same _record_ of allocations across calls to the function with different allocation patterns.
Every call that uses a given _record_, must allocate `Array`s of the same type and size, in the same order.
For example:

```julia
julia> using Test

julia> twos(dims) = 2*ones(dims)
twos (generic function with 1 method)

julia> twos_res, twos_record = record_alloctions(twos, (3,6))
(value = [2.0 2.0 … 2.0 2.0; 2.0 2.0 … 2.0 2.0; 2.0 2.0 … 2.0 2.0], allocation_record = AutoPreallocation.AllocationRecord(
    [Array{Float64,2}(undef, (3, 6)), Array{Float64,2}(undef, (3, 6))],
    [(3, 6), (3, 6)]
))

julia> @test_throws TypeError avoid_alloctions(twos_record, twos, (3,6,9))
Test Passed
      Thrown: TypeError

julia> # If the type is the same and only size differs AutoPreallocation right now won't even
       # error, it will just silently return the wrong result
       avoid_alloctions(twos_record, twos, (30,60))
3×6 Array{Float64,2}:
 2.0  2.0  2.0  2.0  2.0  2.0
 2.0  2.0  2.0  2.0  2.0  2.0
 2.0  2.0  2.0  2.0  2.0  2.0
```

One way to deal with this is to keep track of which parameters change the allocation pattern,
and then declare one _record_ for each of them.

### If a function resizes any array that it allocates you need to call `reinitialize!`
The allocated memory is reuses.
Which means if you call an operation like `push!` or `append!` that resizes it,
then it will be resized the next time it goes to be used.
`reinitialize!(record)` resets the memory to its initial state.

If you are not sure if internally this ever happens then to be safe just call `reinitialize!` everytime before using the _record_.

```julia
julia> function bar(x)
           ys = fill(10.0, 30)
           for ii in 1:100
               push!(ys, x)
           end
           return ys
       end
bar (generic function with 1 method)

julia> @btime bar(3.14);
  684.377 ns (4 allocations: 3.78 KiB)

julia> const _, bar_record = record_alloctions(bar, 3.14);

julia> reinitialize!(bar_record);  # Even the first time after recording

julia> @btime avoid_alloctions($bar_record, bar, 42.0);
  595.262 μs (3 allocations: 64 bytes)

julia> reinitialize!(bar_record);

julia> @btime avoid_alloctions($bar_record, bar, 24601);
  558.079 μs (3 allocations: 64 bytes)
```

### If you are storing output which was allocated within the function, you need to take a `copy`
Due to the memory being reused if the record is used in another function call the output will be over-written.
You should take a `copy` (don't need a `deepcopy`, plain `copy` is fine), to avoid this.

Here is an example of what happens if you don't:
```julia
julia> function mat(x)
           out = zeros(2,2)
           out[1, 1] = x
           out[2, 2] = x
           return out
       end
mat (generic function with 1 method)

julia> const mat1, mat_record = record_alloctions(mat, 1);

julia> (mat1,)
([1.0 0.0; 0.0 1.0],)

julia> mat2 = avoid_alloctions(mat_record, mat, 2);

julia> (mat1, mat2)  # Notice mat1 has changed
([2.0 0.0; 0.0 2.0], [2.0 0.0; 0.0 2.0])

julia> mat3 = avoid_alloctions(mat_record, mat, 3);

julia> (mat1, mat2, mat3)  # Notice: mat1 and mat2 have changed
([3.0 0.0; 0.0 3.0], [3.0 0.0; 0.0 3.0], [3.0 0.0; 0.0 3.0])
```


### AutoPreallocation is not threadsafe
If the function you are recording allocations uses multiple threads, then odds are it will not work with AutoPreallocation.
Recall the requirement that all allocations occur in the same order.
Multithreading disrupts this.

In theory, one could use 1 record per Task.
This is untested.

### Not appropriate for all uses:

 - `avoid_alloctions` makes many small allocations itself #2, so best for if you have large allocations to remove.
 - the _record_ holds in memory all the allocations, thus preventing garbage collection. If you allocate-and-free a ton of memory to run your algorithm then you may run out of RAM.
 - Only handled allocation in the form of `Array` -- which to be fair underlies a great many data structures in Julia.