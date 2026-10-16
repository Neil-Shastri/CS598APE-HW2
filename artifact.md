# Artifact

## How to replicate

We ran everything on the course VM (Intel Xeon Silver 4216 (x86) with 4 cores) inside the Docker image below. Our Makefile uses `-march=native` and an x86-only assembler flag (`-Wa,-mbranches-within-32B-boundaries`), so it needs an x86 machine.

**1. Build the image and start a container** (from the repo root):

```
./dockerbuild.sh neilas3/598ape-hw2
docker run -it --rm --security-opt seccomp=unconfined -v "$(pwd)":/host neilas3/598ape-hw2 /bin/bash
```

The Dockerfile is the course one plus pprof/gperftools/graphviz/valgrind (for profiling).

**2. Build and run** (inside the container, from `/host`, since the program reads `benchmark/*.csv` relative to the current directory):

```
cd /host
make clean && make -j
./genetic_benchmark diabetes > diabetes.rkt && tail -1 diabetes.rkt
./genetic_benchmark cancer > cancer.rkt && tail -1 cancer.rkt
./genetic_benchmark housing > housing.rkt && tail -1 housing.rkt
```

**3. Test a specific optimization.** Check out its commit from the table below **outside** the container (git inside the container complains about the repo owner), then rebuild and rerun inside it like step 2:

```
git checkout <commit>      # outside the container, e.g. git checkout 246a157
make clean && make -j      # inside the container
git checkout main          # outside, to go back
```

## Optimizations

Times are diabetes / cancer / housing in seconds. Speedup is vs the baseline.

| Optimization | Time (s) | Speedup | Commit |
|---|---|---|---|
| Baseline (stock code, -O0) | 20.10 / 128.47 / 240.71 | 1.0x | e97c28e |
| Compile with -O3 | 6.12 / 41.92 / 72.72 | 3.3x / 3.1x / 3.3x | 142472e |
| Add -march=native | 5.06 / 34.70 / 58.97 | 4.0x / 3.7x / 4.1x | cb55adb |
| O(1) stack push/pop | 1.16 / 16.29 / 18.60 | 17.3x / 7.9x / 12.9x | 246a157 |
| Stable sort + branch alignment flag | 1.12 / 10.65 / 18.62 | 17.9x / 12.1x / 12.9x | e8543a5 |
| OpenMP over programs (4 threads) | 0.33 / 3.68 / 5.33 | 61.1x / 34.9x / 45.2x | 2889db9 |

We also tried -O2 (2.0x, same output) and -Ofast (3.8-4.0x), but -Ofast turns on fast-math, which changed the evolved programs for cancer and housing (cancer test accuracy went from 0.947 to 0.894), so we didn't use it.

-flto on top of -O3 was mixed (cancer 6% faster, diabetes and housing 2% slower), so we left it out.

-march=native builds for whatever CPU compiles it. Our VM works well with this flag at about 18% faster than plain -O3 with the same output. On other CPUs the gain may be smaller (we saw none on an M3 Mac).

The stack push/pop used to loop over all 20 slots to find the top of the stack, even though it already keeps a count of how many elements it has. We changed it to just use that count as the index (push/pop are O(1) now). Profiling with debug info -g showed push+pop was about half of the runtime before this change.

The programs get sorted at the end of training with an insertion sort, which copies a whole program every time it shifts one which is N^2 copies, and cancer has 16384 programs, so it ended up being about a third of cancer's time after the stack fix. We switched it to std::stable_sort on indices so each program only gets copied twice (stable so ties stay in the same order).

This made diabetes and housing noticeably slower even though we didn't touch execute_kernel. We figured out execute_kernel just moved to a different address in the binary, and on this intel CPU that matters because of Intel's JCC erratum (https://stackoverflow.com/questions/70620820/why-does-adding-an-if-to-a-c-code-make-it-significantly-faster?utm_source=chatgpt.com). Compiling with -Wa,-mbranches-within-32B-boundaries fixed it. It's an x86 only flag but we only run on the VM.

Almost all the time (94-99%) was now in scoring the programs, and each program gets scored on its own, so we used OpenMP to split the programs across the VM's 4 cores (execute_kernel and the MSE/logLoss loops). Each program still runs on a single thread start to finish, so the output is the same. We used schedule(dynamic) since program lengths vary a lot and it was faster than static. Cancer only gets about 2.9x because more of it is still serial (mostly zero-filling big buffers every generation).
