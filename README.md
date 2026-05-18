# ananicy-cpp Optimized Build

## Goal
Minimize the cache footprint of `ananicy-cpp`, a process scheduler daemon that wakes periodically to scan and re-nice processes. Since it resides in L1 cache while sleeping, every byte of cache it occupies evicts data from performance-critical applications (games, compilation, rendering). The goal is to make it as small as possible to reduce cache pollution.

---

## Optimization Strategy
- **`-Os`** instead of `-O3`/`-O2` — optimize for size, not speed (daemon is IO-bound, sleeps 99% of the time)
- **`-march=native -mtune=native`** — target the build machine's CPU
- **`-flto=full`** — full link-time optimization across all translation units
- **`-fdata-sections -ffunction-sections`** — place each function/data in its own section so `--gc-sections` can strip unused code
- **`--gc-sections --icf=all`** — remove dead code and merge identical functions at link time
- **`-falign-functions=1 -falign-loops=1`** — no alignment padding (every padding byte is a stolen cache line)
- **`-fno-unroll-loops`** — prevents code bloat from loop unrolling
- **`-fomit-frame-pointer`** — one less instruction per function call
- **`-fmerge-all-constants`** — deduplicate identical constants
- **`-fno-plt`** — faster internal function calls, smaller PLT
- **`--strip-all --build-id=none`** — remove all symbols and metadata

---

## Build Fixes Required
The project has issues with newer compilers (missing POSIX headers):
- Added `-include unistd.h` and `-D_GNU_SOURCE` for missing `getpid()` and `mkostemp()` declarations
- CMake's `StandardProjectSettings` and `configure_linker` override user flags, so flags must be forced via `CMAKE_C_FLAGS`/`CMAKE_CXX_FLAGS` set at the end of `CMakeLists.txt`

---

## Results

| Version | .text (code) | Total binary | Cache lines | Alignment |
|---------|-------------|--------------|-------------|-----------|
| **AUR/GIT (standard)** | 603 KB | 610 KB | ~9,648 | 64 bytes |
| **System installed** | 510 KB | 517 KB | ~6,048 | 64 bytes |
| **Optimized build** | **400 KB** | **408 KB** | **~4,400** | **16 bytes** |

---

## Improvement over AUR/GIT
- **-34%** smaller .text section
- **~5,200 fewer cache lines** evicted per wake cycle

---

## Improvement over system installed
- **-22%** smaller .text section
- **~1,600 fewer cache lines** evicted per wake cycle

---

## Practical Impact
Each time ananicy-cpp wakes (every few seconds by default), it pulls its code into L1 instruction cache. With the optimized build:
- **Less cache eviction** of the currently running heavy process
- **Lower latency** when returning to the foreground application
- **Reduced micro-stutter** in latency-sensitive workloads like gaming

---

## Flags Used
```Fish
# Common flags
-include unistd.h -D_GNU_SOURCE
-march=native -mtune=native
-Os -pipe -fno-plt
-fmerge-all-constants -fomit-frame-pointer
-fno-unroll-loops
-falign-functions=1 -falign-loops=1
-fno-math-errno -fno-trapping-math
-fstack-protector -D_FORTIFY_SOURCE=1
-fdata-sections -ffunction-sections
-flto=full
```

# Linker flags
```Fish
-flto=full -fuse-ld=lld
-Wl,-O3,--relax,--gc-sections,--strip-all
-Wl,--lto-O3
-Wl,--lto-whole-program-visibility
-Wl,--icf=all
-Wl,-z,now -Wl,-z,relro
-Wl,--build-id=none
```
