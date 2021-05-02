# Perf MacOS

This is an effort to bring userland hardware perf-counters to MacOS as a single header c++ interface library. Hardware
perf counters are usually subject to tight permission restrictions, i.e., may only be executed on ring-0. Therefore,
Kernel level support is usually required to access hardware performance counters.

One known route for perf support is to write and load a KEXT which provides access to these instructions. Since this
might negatively affect system stability and is generally not userfriendly, **this project does not rely on a KEXT.**
Instead, it is based on the XNU perf counter implementation. Some relevant files include:

* [pmc.h](https://opensource.apple.com/source/xnu/xnu-2050.18.24/osfmk/pmc/pmc.h.auto.html).
* [arm64 kpc.c](https://opensource.apple.com/source/xnu/xnu-4570.1.46/osfmk/arm64/kpc.c.auto.html)
* [x86_64 kpc.c](https://opensource.apple.com/source/xnu/xnu-4570.1.46/osfmk/x86_64/kpc_x86.c.auto.html)
* ... (for more info, see [opensource.apple.com](https://opensource.apple.com/))

To be able to access these kernel features, perf-macos uses the first party **private**
Framework [kperf](http://newosxbook.com/src.jl?tree=xnu&file=/osfmk/kperf/kperf.h).

Note that relying on the private kperf framework and undocumented kernel APIs means that this code could break at any
point without further notice. Please also acknowledge that submitting Apps to the MacOS Appstore that contain this code
will likely result in a rejection, due to the use of private APIs. Afaik, Apple could even count your app submission as
attempted fraud.

# Usage

For full usage examples, see
[test.cpp](https://github.com/DominikHorn/perf-macos/blob/main/test.cpp).

### Basic Usage

```c++
#include "perf-macos.hpp"

// ...

const uint64_t n = 1000000;

// Initialize counter. This will take care of setting everything up for perf measurements
Perf::Counter counter;

// Start measuring
counter.start();

// Code to benchmark. Iterated n-times to get accurate measurements
for (uint64_t i = 0; i < n; i++) {
    const auto val = 0xABCDEF03 / (i + 1);
    
    // See discussion for explanation
    DoNotEliminate(val);
}

// Stop measuring
auto measurement = counter.stop();

// Average measurements for our iterations and pretty print
measurement.averaged(n).pretty_print();
```

### BlockCounter

```c++
#include "perf-macos.hpp"

// ...

const uint64_t n = 1000000;

{
    // This will automatically start() after construction and stop() on destruction
    Perf::BlockCounter b(n);

    // Code to benchmark. Iterated n-times to get accurate measurements
    for (uint64_t i = 0; i < n; i++) {
        const auto val = i ^ (i + 0xABCDEF01);
        
        // See discussion for explanation
        DoNotEliminate(val);
    }
}
```

## Output

Benchmarking `x ^ (x + 0xABCDEF01)` yields the following sample output on my machine:

```
   Elapsed [ns]   Instructions  Branch misses      L1 misses     LLC misses
       0.682419       5.000810       0.000014       0.000002       0.000026
```

Note that elapsed time will vary across runs. Please also note that the tested CPU only has 4 configurable perf counter
registers and therefore only 4 concurrent measurements were taken.

## DoNotEliminate - Discussion

Benchmarking code contains `DoNotEliminate(x)` statements to prevent the compiler from removing the computation we want to 
benchmark during optimization, or even the entire loop for that matter. One commonly used method to prevent this from happening
is to accumulate the computation result into some variable and print that variable to `std::cout` after the loop. It seems this 
alone is not sufficient however to ensure that the compiler does not perform unwanted optimizations/eliminations.

`DoNotEliminate(x)` effectively tags the computation leading to the result `x` as having some observable magic side
effect. This in turn prohibits any fancy tricks to get around computing `x`, therefore ensuring that the emitted assembly
always exactly contains the expected computation. For an explanation of how this works,
see [this video](https://www.youtube.com/watch?v=nXaxk27zwlk&t=2441s, improved version from). The version in use
in [test.cpp](https://github.com/DominikHorn/perf-macos/blob/main/test.cpp) stems
from [google benchmark](https://github.com/google/benchmark/blob/ba9a763def4eca056d03b1ece2946b2d4ef6dfcb/include/benchmark/benchmark.h#L326)
, and is defined as follows:

```c++
#define DoNotEliminate(value) asm volatile("" : : "r,m"(value) : "memory")
```

# Alternatives

Consider using the official "Counters" instrument template from the Instruments App, which is currently delivered as
part of the [Xcode IDE](https://developer.apple.com/xcode/features/).

