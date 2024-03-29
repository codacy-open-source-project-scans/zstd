# Fuzzing

Each fuzzing target can be built with multiple engines.
Zstd provides a fuzz corpus for each target that can be downloaded with
the command:

```
make corpora
```

It will download each corpus into `./corpora/TARGET`.

## fuzz.py

`fuzz.py` is a helper script for building and running fuzzers.
Run `./fuzz.py -h` for the commands and run `./fuzz.py COMMAND -h` for
command specific help.

### Generating Data

`fuzz.py` provides a utility to generate seed data for each fuzzer.

```
make -C ../tests decodecorpus
./fuzz.py gen TARGET
```

By default it outputs 100 samples, each at most 8KB into `corpora/TARGET-seed`,
but that can be configured with the `--number`, `--max-size-log` and `--seed`
flags.

### Build
It respects the usual build environment variables `CC`, `CFLAGS`, etc.
The environment variables can be overridden with the corresponding flags
`--cc`, `--cflags`, etc.
The specific fuzzing engine is selected with `LIB_FUZZING_ENGINE` or
`--lib-fuzzing-engine`, the default is `libregression.a`.
Alternatively, you can use Clang's built in fuzzing engine with
`--enable-fuzzer`.
It has flags that can easily set up sanitizers `--enable-{a,ub,m}san`, and
coverage instrumentation `--enable-coverage`.
It sets sane defaults which can be overridden with flags `--debug`,
`--enable-ubsan-pointer-overflow`, etc.
Run `./fuzz.py build -h` for help.

### Running Fuzzers

`./fuzz.py` can run `libfuzzer`, `afl`, and `regression` tests.
See the help of the relevant command for options.
Flags not parsed by `fuzz.py` are passed to the fuzzing engine.
The command used to run the fuzzer is printed for debugging.

Here's a helpful command to fuzz each target across all cores,
stopping only if a bug is found:
```
for target in $(./fuzz.py list); do
    ./fuzz.py libfuzzer $target -jobs=10 -workers=10 -max_total_time=1000 || break;
done
```
Alternatively, you can fuzz all targets in parallel, using one core per target:
```
python3 ./fuzz.py list | xargs -P$(python3 ./fuzz.py list | wc -l) -I__ sh -c "python3 ./fuzz.py libfuzzer __ 2>&1 | tee __.log"
```
Either way, to double-check that no crashes were found, run `ls corpora/*crash`.
If any crashes were found, you can use the hashes to reproduce them.

## LibFuzzer

```
# Build the fuzz targets
./fuzz.py build all --enable-fuzzer --enable-asan --enable-ubsan --cc clang --cxx clang++
# OR equivalently
CC=clang CXX=clang++ ./fuzz.py build all --enable-fuzzer --enable-asan --enable-ubsan
# Run the fuzzer
./fuzz.py libfuzzer TARGET <libfuzzer args like -jobs=4>
```

where `TARGET` could be `simple_decompress`, `stream_round_trip`, etc.

### MSAN

Fuzzing with `libFuzzer` and `MSAN` is as easy as:

```
CC=clang CXX=clang++ ./fuzz.py build all --enable-fuzzer --enable-msan
./fuzz.py libfuzzer TARGET <libfuzzer args>
```

`fuzz.py` respects the environment variables / flags `MSAN_EXTRA_CPPFLAGS`,
`MSAN_EXTRA_CFLAGS`, `MSAN_EXTRA_CXXFLAGS`, `MSAN_EXTRA_LDFLAGS` to easily pass
the extra parameters only for MSAN.

## AFL

The default `LIB_FUZZING_ENGINE` is `libregression.a`, which produces a binary
that AFL can use.

```
# Build the fuzz targets
CC=afl-clang CXX=afl-clang++ ./fuzz.py build all --enable-asan --enable-ubsan
# Run the fuzzer without a memory limit because of ASAN
./fuzz.py afl TARGET -m none
```

## Regression Testing

The regression test supports the `all` target to run all the fuzzers in one
command.

```
CC=clang CXX=clang++ ./fuzz.py build all --enable-asan --enable-ubsan
./fuzz.py regression all
CC=clang CXX=clang++ ./fuzz.py build all --enable-msan
./fuzz.py regression all
```

## Fuzzing a custom sequence producer plugin
Sequence producer plugin authors can use the zstd fuzzers to stress-test their code.
See the documentation in `fuzz_third_party_seq_prod.h` for details.

## Adding a new fuzzer
There are several steps involved in adding a new fuzzer harness.

### Build your harness
1. Create a new your fuzzer harness `tests/fuzz/your_harness.c`.

2. Add your harness to the Makefile

    2.1 Follow [this example](https://github.com/facebook/zstd/blob/e124e39301381de8f323436a3e4c46539747ba24/tests/fuzz/Makefile#L216) if your fuzzer requires both compression and decompression symbols (prefix `rt_`). If your fuzzer only requires decompression symbols, follow [this example](https://github.com/facebook/zstd/blob/6a0052a409e2604bd40354b76b86272b712edd7d/tests/fuzz/Makefile#L194) (prefix `d_`).
    
    2.2 Add your target to [`FUZZ_TARGETS`](https://github.com/facebook/zstd/blob/6a0052a409e2604bd40354b76b86272b712edd7d/tests/fuzz/Makefile#L108).
    
3. Add your harness to [`fuzz.py`](https://github.com/facebook/zstd/blob/6a0052a409e2604bd40354b76b86272b712edd7d/tests/fuzz/fuzz.py#L48).

### Generate seed data
Follow the instructions above to generate seed data:
```
make -C ../tests decodecorpus
./fuzz.py gen your_harness
```

### Run the harness
Follow the instructions above to run your harness and fix any crashes:
```
./fuzz.py build your_harness --enable-fuzzer --enable-asan --enable-ubsan --cc clang --cxx clang++
./fuzz.py libfuzzer your_harness
```

### Minimize and zip the corpus
After running the fuzzer for a while, you will have a large corpus at `tests/fuzz/corpora/your_harness*`.
This corpus must be minimized and zipped before uploading to GitHub for regression testing:
```
./fuzz.py minimize your_harness
./fuzz.py zip your_harness 
```

### Upload the zip file to GitHub
The previous step should produce a `.zip` file containing the corpus for your new harness.
This corpus must be uploaded to GitHub here: https://github.com/facebook/zstd/releases/tag/fuzz-corpora


