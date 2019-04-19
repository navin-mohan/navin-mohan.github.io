---
title: "Playing around with OpenMP and C++ - Parallelizing for-loops"
date: 2019-04-19
excerpt: "Optimizing a toy C++ library using OpenMP"
---

Yesterday, I came across [an interesting repository](https://github.com/georgeshanti/cpp-transformations) on my GitHub feed. It is a simple and functional header-only C++ library that lets you apply transformations like map, reduce and filter to standard C++ containers.

Well, yes I'm aware of the fact that it is possible to perform all three tranformations using just STL.

* Map - [`std::transform`](http://www.cplusplus.com/reference/algorithm/transform/)
* Filter - [`std::copy_if`](http://www.cplusplus.com/reference/algorithm/copy_if/)
* Reduce - [`std::accumulate`](http://www.cplusplus.com/reference/numeric/accumulate/)

This [stackoverflow answer](https://stackoverflow.com/questions/40901615/how-to-replicate-map-filter-and-reduce-behaviors-in-c-using-stl) explains them in detail.

But that is not why I was interested. Before getting in to that let's take a brief digression.

## The Beauty of Functions

If you're anything like me and have taken the effort to go through the horrors of functional programming, then you must know these functions(map,reduce,filter) from another context.

<p align="center">
    <img src="/static/img/monad-meme.jpg" alt="Monads"/>
</p>

They are called higher order functions, which is just a fancy name for functions that take functions as arguments(or returns one). 

One thing that is most celebrated in the land of functional programming is , of course, functions and their inherent lack of side effects. The key observation here is that such functions(pure functions) are independent of thier order of execution as they don't have any side effects. Just keep that in mind as we will be needing it soon.

## The Problem

Coming back to the reason why I got interested in the first place. The library seemed to work fine and it was all generic code but appeared to slow down with large input sizes. The bottleneck was found to be a serial for-loop which is central.

This was [the point](https://github.com/nvnmo/cpp-transformations/tree/53714dd397b62c25b1ac9961562beb2a59154425) in the commit history where I started.

But as we have seen in the previous section, if the functions are pure then we don't have to apply them in a serial order. Which means we are free to utilze the hardware parallelism while applying those functions.

So it got me wondering if there is a better way. Somehow making that for-loop run parallel would solve the issue. Which is exactly what I did.

This post is all about the path I took to get a speed up of ~2x on my machine.

## OpenMP and Parallel Programming
After some research, it was clear that [OpenMP](https://www.openmp.org) is what I was looking for. It supports C++ through GCC and can be easily enabled by using the `pragma omp` directives when needed.

The best part is that it can parallelize the for-loop with very minimal changes to the original code. We'll get into the details in a later section.

## Benchmarking code
First things first, we don't want to manually compile everything whenever something changes. That's why build systems exist so let's use one.

I will be using [CMake](https://cmake.org/) here. If you're new to CMake I recommend reading [this blog post](https://pabloariasal.github.io/2018/02/19/its-time-to-do-cmake-right/).

To perform the actual benchmarking let's just use the `high_resolution_clock` from the standard chrono library. 

```cpp
/*
** test_vec is a large std::vector<double> containing 
** random numbers
*/

// start time 
auto start_map = std::chrono::high_resolution_clock::now();

// runs the actual transformation
transform::map<double, decltype(test_vec)>(test_vec, 
test_map_func);

// end time
auto end_map = std::chrono::high_resolution_clock::now();

// the runtime duration
auto duration = end_map - start_map;
```

We'll be doing this for all three functions and then prints out the results.

Now that the benchmarking code is out of the way, let's focus on the CMake file.

We need two targets here, a parallel version and a non-parallel version. So that we can see if we're making any progress.

Let's define the non parallel target first.

```cmake
# add out target
add_executable(benchmark-no-parallel benchmark.cpp)

# add the source director as an include directory
target_include_directories(benchmark-no-parallel PRIVATE ${CMAKE_SOURCE_DIR})

# nothing fancy here
target_compile_options(benchmark-no-parallel PRIVATE -Wall)

# finally run our target to see the results
add_custom_command(
    TARGET benchmark-no-parallel POST_BUILD
    COMMAND ./benchmark-no-parallel
    WORKING_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
    COMMENT "Running benchmarks on non parallel version..."
)
```

Since we're using OpenMP all we need to change is the compile options a tad bit. But before that we need to check if we actually have OpenMP available on the system. Luckily, CMake can take care of that in a few lines.

```cmake
# check for OpenMP
find_package(OpenMP)

# if OpenMP is available
if(OPENMP_FOUND)
    add_executable(benchmark-parallel benchmark.cpp)
    target_include_directories(benchmark-parallel PRIVATE ${CMAKE_SOURCE_DIR})
    
    # include the OpenMP compile flags as well
    target_compile_options(benchmark-parallel PRIVATE ${OpenMP_CXX_FLAGS})
    
    # add the OpenMP libraries for linking
    target_link_libraries(benchmark-parallel PRIVATE ${OpenMP_CXX_LIBRARIES})

    # finally run our target to see the results
    add_custom_command(
        TARGET benchmark-parallel POST_BUILD
        COMMAND ./benchmark-parallel
        WORKING_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
        COMMENT "Running benchmarks on parallel version..."
    )

# if OpenMP is not available
else()
    message(WARNING "OpenMP not found. Cannot run benchmarks without it")
endif()
```

And that's it, we have our build system ready. Now if we run it we'll be able to see the benchmarks being run for both the targets.

We have not yet added OpenMP directives so the results will be more or less the same.

## Parallelizing the for-loop
As odd as it may seem, this is the easiest part of all. All we need is a little restructuring of the loop construct and the openmp pragma directive.


Starting with the map function.

```cpp
/*
** The Map function
*/


// non parallel
auto new_arr = container();
for(auto it=arr.begin(); it!=arr.end(); it++){
    auto new_ele = func(*it);
    new_arr.insert(new_arr.end(), new_ele);
}

// parallel
auto new_arr = container(arr.size());
size_t len = arr.size();
auto it = arr.begin();
auto new_it = new_arr.begin();

// openmp magic
#pragma omp parallel for schedule(static)
for(size_t i = 0; i < len; i++){
    auto new_ele = func(*(it+i));
    *(new_it+i) = new_ele;
}

```

You can find more details on the scheduling part [here](https://stackoverflow.com/questions/10850155/whats-the-difference-between-static-and-dynamic-schedule-in-openmp).

Let's do filter function.

```cpp
/*
** The Filter function
*/

// non parallel
auto new_arr = container();
for(auto it=arr.begin(); it!=arr.end(); it++){
    if(func(*it))
        new_arr.insert(new_arr.end(), *it);
}

// parallel
auto new_arr = container();
size_t len = arr.size();
auto it = arr.begin();
#pragma omp parallel for schedule(static)
for(size_t i = 0; i < len; i++){
    if(func(*it)){
        /* the following block is a critical section
        ** we have to tell openmp that no
        ** two threads can access it at the same time
        */
        #pragma omp critical                
        {new_arr.insert(new_arr.end(), *(it+i));}
    }
}
```

Lastly, the reduce function. This one's a bit tricky. As we discussed earlier the function application remains side-effect free can be ran in parallel but the actual reduction part will take at least $$log n$$ serial steps for $$n$$ results. Since we're assuming that the function application is the most expensive part of the task, the reduction overhead should be outweighed by the overall savings.

Luckily, OpenMP makes it very easy to use reductions. You can read more about reductions [here](http://pages.tacc.utexas.edu/~eijkhout/pcse/html/omp-reduction.html).

```cpp
/*
** The Reduce function
*/

// non parallel
for(auto it=arr.begin(); it!=arr.end(); it++){
    initial += func(*it);
}

// parallel
size_t len = arr.size();
auto it = arr.begin();
/*
** Tells openmp to apply a sum reduction on
** the variable initial
*/
#pragma omp parallel for schedule(static) reduction(+:initial)
for(size_t i = 0; i < len; i++){
    initial += func(*(it+i));
}
```


And that is it. We have parallelized the for-loops. Let's run a benchmark test.

### Normal for-loop
```text
Results
Map   : 418ms
Reduce: 178ms
Filter: 109ms
```

### Parallel for-loop
```text
Results
Map   : 216ms
Reduce: 85ms
Filter: 55ms
```

The tests were performed on my dual core machine with a vector of 10 million double values. Our optimization nearly slashed the runtimes to half the original.

You can find the code [here](https://github.com/nvnmo/cpp-transformations).