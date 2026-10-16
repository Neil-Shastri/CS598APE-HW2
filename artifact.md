# Artifact

## Setup

Build the image and start a container from the repo root:

```
./dockerbuild.sh neilas3/598ape-hw2
docker run -it --rm --security-opt seccomp=unconfined -v "$(pwd)":/host neilas3/598ape-hw2 /bin/bash
```

We added pprof, graphviz, gperftools and valgrind to the end of the Dockerfile. Everything else is the same as the course one.

We ran everything on the course VM (4 cores) with g++ 13.3.0 from the image.

## Running

Inside the container, from `/host` (the program loads `benchmark/*.csv` relative to the current directory):

```
make clean && make -j
./genetic_benchmark diabetes | tail -1
./genetic_benchmark cancer | tail -1
./genetic_benchmark housing | tail -1
```

The time we report is the `;;Time(...)` line at the end of each run.

## Evaluating an optimization

Check out the commit from the table (outside the container), then rebuild and run inside it:

```
git checkout <commit>
make clean && make -j && ./genetic_benchmark diabetes | tail -1
```

To check the output didn't change, save the output of the baseline commit and diff it against the new one, ignoring the time line:

```
diff <(grep -v '^;;Time(' base.rkt) <(grep -v '^;;Time(' new.rkt)
```

## Profiling

```
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libprofiler.so.0 CPUPROFILE=out.prof ./genetic_benchmark diabetes > /dev/null
/root/go/bin/pprof -http=0.0.0.0:8080 -no_browser ./genetic_benchmark out.prof
```

## Optimizations

Times are diabetes / cancer / housing in seconds. Speedup is vs the baseline.

| Optimization | Time (s) | Speedup | Commit |
|---|---|---|---|
| Baseline (stock code, -O0) | 20.10 / 128.47 / 240.71 | 1.0x | e97c28e |
| Compile with -O3 | 6.12 / 41.92 / 72.72 | 3.3x / 3.1x / 3.3x | 142472e |
| Add -march=native | 5.06 / 34.70 / 58.97 | 4.0x / 3.7x / 4.1x | cb55adb |
| O(1) stack push/pop | 1.16 / 16.29 / 18.60 | 17.3x / 7.9x / 12.9x | 246a157 |
| Stable sort + branch alignment flag | 1.12 / 10.65 / 18.62 | 17.9x / 12.1x / 12.9x | TBD |

We also tried -O2 (2.0x, same output) and -Ofast (3.8-4.0x), but -Ofast turns on fast-math, which changed the evolved programs for cancer and housing (cancer test accuracy went from 0.947 to 0.894), so we didn't use it.

-flto on top of -O3 was mixed (cancer 6% faster, diabetes and housing 2% slower), so we left it out.

-march=native builds for whatever CPU compiles it. Our VM works well with this flag at about 18% faster than plain -O3 with the same output. On other CPUs the gain may be smaller (we saw none on an M3 Mac).

The stack push/pop used to loop over all 20 slots to find the top of the stack, even though it already keeps a count of how many elements it has. We changed it to just use that count as the index (push/pop are O(1) now). Profiling with debug info -g showed push+pop was about half of the runtime before this change.

The programs get sorted at the end of training with an insertion sort, which copies a whole program every time it shifts one which is N^2 copies, and cancer has 16384 programs, so it ended up being about a third of cancer's time after the stack fix. We switched it to std::stable_sort on indices so each program only gets copied twice (stable so ties stay in the same order).

This made diabetes and housing noticeably slower even though we didn't touch execute_kernel. We figured out execute_kernel just moved to a different address in the binary, and on this intel CPU that matters because of Intel's JCC erratum (https://stackoverflow.com/questions/70620820/why-does-adding-an-if-to-a-c-code-make-it-significantly-faster?utm_source=chatgpt.com). Compiling with -Wa,-mbranches-within-32B-boundaries fixed it. It's an x86 only flag but we only run on the VM.
