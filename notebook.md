# PDX

Setting up [PDX](https://github.com/cwida/PDX).

## Preliminaries

Setup the `direnv` including a python setup.

Install GFlags (formerly Google Commandline Flags) and the setuptools:

    sudo apt install libgflags-dev
    pip install setuptools

Intel's MKL gave some issues, but I managed as follows:

    sudo apt install intel-mkl-full

# For running the tests shipped with PDX on the Ubuntu machine I at some point needed (but no longer?):
# `export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libmkl_def.so:/usr/lib/x86_64-linux-gnu/libmkl_avx2.so:/usr/lib/x86_64-linux-gnu/libmkl_core.so:/usr/lib/x86_64-linux-gnu/libmkl_intel_lp64.so:/usr/lib/x86_64-linux-gnu/libmkl_intel_thread.so:/usr/lib/x86_64-linux-gnu/libiomp5.so`

## PDX

From the PDX docs, install the following dependencies:

    pip install swig
    pip install -r requirements.txt
    python setup.py clean --all
    python -m pip install .

We also need FAISS, and will reuse the version that is checked out already
through the DuckDB FAISS extension:

    cd duckdb-faiss-ext/faiss/

Before building the version needed for the PDX tests, overrule its Python Environment,
so the results will be installed in PDX's venv (instead of the extension's one):

    export VIRTUAL_ENV=/home/arjen/gh/PDX/.venv
    export PATH=/home/arjen/gh/PDX/.venv/bin:$PATH

Now build the FAISS package for python from source:

    cmake -DFAISS_ENABLE_PYTHON=ON \
          -DCMAKE_BUILD_TYPE=Release -DFAISS_OPT_LEVEL=avx512 \
          -DBLA_VENDOR=Intel10_64_dyn -DMKL_LIBRARIES=/usr/lib/x86_64-linux-gnu/mkl/* \
          -B build .
    make -C build -j faiss
    make -C build -j swigfaiss

And install it into the PDX virtual environment:

    (cd build/faiss/python && python setup.py install)

Run the examples, e.g.

    python ./examples/pdxearch_simple.py
    python examples/pdxearch_exact_bond.py 

Benchmarking

The repo provides [instructions for benchmarking](BENCHMARKING.md).

Build the benchmarking code:

    cmake . -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_FLAGS_RELEASE="-O3 -mprefer-vector-width=512" 

Tusi is an Intel [Cascade Lake](https://www.intel.com/content/www/us/en/products/platforms/details/cascade-lake.html)
machine with 2 [Intel(R) Xeon(R) Silver 4214 CPU @ 2.20GHz](https://openbenchmarking.org/s/2+x+Intel+Xeon+Silver+4214) 
CPUs with 24 cores each, so we should compile for AVX-512.

Download the data.
Modify `python_scripts/setup_settings.py` to specify the desired experiments.

Prepare the data:

    pip install -r ./benchmarks/python_scripts/requirements.txt
    python ./benchmarks/python_scripts/setup_data.py > log/setup_data.log

Run the experiments:

    benchmarks/BenchmarkPDXBOND "sift-128-euclidean" 1
    benchmarks/BenchmarkPDXBOND "contriever-768" 1
    benchmarks/BenchmarkPDXBOND "openai-1536-angular" 1

The second parameter refers to the dimension ordering 
(referring to this enum in `include/pdx/pdxearch.hpp`):

```
enum PDXearchDimensionsOrder {
    SEQUENTIAL,
    DISTANCE_TO_MEANS,
    DECREASING,
    DISTANCE_TO_MEANS_IMPROVED,
    DECREASING_IMPROVED,
    DIMENSION_ZONES
};
```

Run a series of experiments:

    ( for d in sift-128-euclidean msong-420 contriever-768 openai-1536-angular
      do 
        echo Experiment $d @ `date` 
        benchmarks/BenchmarkPDXBOND $d 1 
      done > benchmarks-bond.log & 
    ) &

What's in the datasets?

```
export PATH=$HOME/RU/duckdb-faiss-ext/build/release:$PATH
duckdb <<__EOSQL__
.echo ON
install hdf5 from community;
load hdf5;
describe FROM read_hdf5("benchmarks/datasets/downloaded/msong-420.hdf5", "train");
describe FROM read_hdf5("benchmarks/datasets/downloaded/msong-420.hdf5", "test");
__EOSQL__
```

    h5ls benchmarks/datasets/downloaded/msong-420.hdf5 
