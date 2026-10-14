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
| Compile with -O3 | 6.12 / 41.92 / 72.72 | 3.3x / 3.1x / 3.3x | TBD |

We also tried -O2 (2.0x, same output) and -Ofast (3.8-4.0x), but -Ofast turns on fast-math, which changed the evolved programs for cancer and housing (cancer test accuracy went from 0.947 to 0.894), so we didn't use it.
