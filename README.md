# PGA

This repository contains the implementation of Proximal Gradient Analysis (PGA). PGA is implemented on a modified version of LLVM 7.0. Please see our paper, Fine Grained Dataflow Analysis with Proximal Gradients ([link](https://arxiv.org/pdf/1909.03461.pdf)) for more details.

## Building from Source
The dependency list and instructions to build PGA are below. 

### Dependencies
On ubuntu 18.04: 
```
sudo apt install make cmake build-essential bison ninja-build zlib1g-dev autoconf m4 texinfo flex
```

### Build LLVM
```
mkdir build
cd build
../configure.sh
cmake --build . --target install 2>&1 | tee build.log
```

## Building from Docker
To obtain the docker image, you can:
1. Pull from Dockerhub via `docker pull abhishekshah212/pga`
2. Build the image locally via `docker build -t pga .`

Then, run the docker container with

```
docker run -it pga
# or if you pulled from Dockerhub, use docker run -it abhishekshah212/pga 
```

## Usage
PGA is implemented in a modified version of LLVM's taint tracking framework, DataFlowSanitizer (`dfsan`). Gradients can be tracked by:
1. modifying the source
2. marking input bytes from file reads


### (1) Modifying Source

To track gradient by modifying source code, use include `#include <sanitizer/dfsan_interface.h>`  and add the following to mark a variable in the file:
```
dfsan_label x_label = dfsan_create_label("x");
dfsan_set_label(x_label, &x, sizeof(x));
```
This creates an initial directional derivative with directions +1, -1 on variable `x`. To obtain the gradient of another variable `y` wrt. `x`, get the label of `y` and look up its gradient metadata:
```
dfsan_label y_label = dfsan_get_label(y);
const struct dfsan_label_info* y_info = dfsan_get_label_info(y_label);
printf("y label %d: %f, %f\n",y_label, y_info->neg_dydx, y_info->pos_dydx);
```

To build use the flag `-fsanitize=dataflow` when compiling. A small example program and makefile are included under `example/` that demonstrate marking and reading gradients, as well as generating the llvm bitcode during compilation. To build the small example program, use the following commands:

```
cd example
make
DFSAN_OPTIONS="branch_logfile=branches.csv" ./test_int.exe
cd ..
```
The output shows which variables have gradients wrt to the input `x`. 
```
x     label 2: 0.000000, 1.000000
y=x*4 label 3: 0.000000, 4.000000
z=y(mod)4 label 4: 0.000000, 0.000000
loop  label 7: 0.000000, 96.000000
```

You can also find two new files in the `example` directory after running `./test_int.exe`:
1. `gradient.csv` which contains intermediate gradients from instructions wrt. to `x`.  
2. `branches.csv` which contains recorded gradients wrt. to `x` on intermediate branch variables. 

```
# contents of gradient.csv
label,ndx,pdx,location,f_val,opcode
1,1.000000,1.000000,fread auto,0,
2,0.000000,1.000000,test_int.c:13,0,
3,0.000000,4.000000,test_int.c:14,4,Mul
4,0.000000,0.000000,test_int.c:15,0,SRem
5,0.000000,8.000000,test_int.c:19,8,Mul
6,0.000000,24.000000,test_int.c:19,24,Mul
7,0.000000,96.000000,test_int.c:19,96,Mul

# contents of branches.csv
file_id,inst_id,lhs_label,rhs_label,lhs_val,rhs_val,lhs_ndx,lhs_pdx,rhs_ndx,rhs_pdx,cond_val,zero,is_ptr,location
438997255675854297,0,2,0,1.000000,0.000000,1.000000,1.000000,0.000000,0.000000,1,0,0,test_int.c:13
```

### (2) Building Real-World Programs and Tracking File Reads

To build a target with PGA, use the clang built in `build/bin/clang` and include `-fsanitize=dataflow` in the CFLAGS. For example, binutils can be built as follows:
```
wget https://gnu.askapache.com/binutils/binutils-2.36.tar.gz
tar xzvf binutils-2.36.tar.gz
cd binutils-2.36
mkdir build; cd build;

LLVM_BINPATH="`pwd`/../../build/bin/"
CC="${LLVM_BINPATH}/clang" \
NM="${LLVM_BINPATH}/llvm-nm" \
AR="${LLVM_BINPATH}/llvm-ar" \
OBJDUMP="${LLVM_BINPATH}/llvm-objdump" \
RANLIB="${LLVM_BINPATH}/llvm-ranlib" \
CFLAGS="-fsanitize=dataflow" \
../configure

make -j16
```

A particular byte read from a file can be automatically tracked by setting the environment variable `FREAD_BYTE_IDX=<byte_idx>`. For example, given the prior ELF file `test_int.exe`, file byte 100 can be automatically tracked when running objdump, and any recorded derivatives on branches from byte 100 logged to `branches.csv`. By default all derivatives will also be logged to `gradient.csv`.
```
FREAD_BYTE_IDX=100 DFSAN_OPTIONS="branch_logfile=branches.csv" ./binutils/objdump -xD ../../example/test_int.exe >/dev/null
```



### Options

Options are set as a comma separated list in the environment variable `DFSAN_OPTIONS`. Options are specified in the `dfsan_flags.inc` file, relevant options from the flags file are shown below in the format `type, option_name, default, description`:

```
DFSAN_FLAG(const char *, gradient_logfile, "gradient.csv", 
"Log file for gradient (recorded as csv).")

DFSAN_FLAG(const char *, branch_logfile, "", 
"Log file for branch gradients (recorded as csv).")

DFSAN_FLAG(const char *, func_logfile, "",
"Log file for function gradients (recorded as csv).")

DFSAN_FLAG(bool, reuse_labels, true, 
"Optimization to reuse labels when gradient does not change")

DFSAN_FLAG(int, samples, 5,
"Samples to use with most ops.")

DFSAN_FLAG(bool, branch_barriers, 1, 
"Use barrier functions with branches.")

DFSAN_FLAG(int, gep_default, 1,
"Default value for GEPs with nonzero grad inputs")

DFSAN_FLAG(int, select_default, 1,
"Default value for Selects with nonzero grad inputs")

DFSAN_FLAG(bool, default_nan, false, "Default to nan for unsupported ops.")
```

For example to use 10 samples and branch barriers, the options can be set:
```
DFSAN_OPTIONS="samples=10,branch_barriers=1"  <program cmd>
```

Note that with the default `branch_barriers` enabled, some gradients will be set to 0 depending on the execution path branch constraints. Set `branch_barriers=0` to disable this behavior.
