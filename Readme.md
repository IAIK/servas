**Disclaimer**: This is not a self-running repo. 
It contains patches for the SERVAS prototype used for performance evaluation. 
It containts the RISC-V hardware and a number of software components and containes the respective commits as submodules. 
This is a research prototype used to evaluate the performance of the involved components.


# Organisation
```
hardware/
     cva6/                SERVAS hardware prototype
patches/                  Contains all the patches for the submodule repos
software/
     ariane-sdk/          Builds a bootable system image for the cva6 hardware.
     riscv-isa-sim/       Spike ISA Simulator
     riscv-opcodes/       Generates configuration files for instructions/CSRs/exception
     riscv-pk/            Proxykernel and Security Monitor
     riscv-tests/         Testcases for RISC-V and SERVAS
```

# Dependencies
Please, consult the respective submodules for the required dependencies.

# Instructions
First, clone the repo and initialize the first layer of submodules. 
```
git clone 
cd servas
BASE=$(pwd)
git submodule update --init
```

From the topmost directory you can then apply the patches and initialize the remaining submodules:
```
git -C ${BASE:?}/hardware/cva6 am  ../../patches/cva6-0001-Add-SERVAS-support.patch
git -C ${BASE:?}/software/ariane-sdk am  ../../patches/ariane-sdk-0001-Add-SERVAS-support.patch
git -C ${BASE:?}/software/riscv-opcodes am  ../../patches/riscv-opcodes-0001-Add-SERVAS-support.patch
git -C ${BASE:?}/software/riscv-tests am  ../../patches/riscv-tests-0001-Add-SERVAS-testcases.patch
git -C ${BASE:?}/software/riscv-isa-sim am  ../../patches/riscv-isa-sim-0001-Add-SERVAS-support.patch
git -C ${BASE:?}/software/riscv-pk am  ../../patches/riscv-pk-0001-Add-SERVAS-support.patch
git submodule foreach "git submodule update --init --recursive"
```


# Building
## Hardware
The evaluation protoype's hardware design can be found in `hardware/cva6`. 
The patch adds preliminary support for SERVAS' logic, registers and the interface to the memory encryption engine to evaluate the system's performance and hardware overhead. 
The tweak is supposed to be supplied via AXI-user signals, which is [currently not supported in the cva6 core](https://github.com/openhwgroup/cva6/issues/550).
In the `fpga/include` there are two defines to activate the logic behind RVAS with he `RVAS` define and the memory encryption via the `MEE` define.
For more details on how to run cva6, consult the repo's manual.
**Note:** Our hardware prototype implementation was only used for the performance evaluation but currently does not provide any security guarantees. This is due to bugs in the Memsec core (missing authentication exceptions), missing user signals in cva6, and missing synchronization of tweaks between Memsec and the caches. These limitations, however, should not impact the performance evaluation but only become relevant in case of security violations. Our spike/software implementation accurately models the functional logic (isolation guarantees).

To build a memory configuration file for the Xilinx KC705 board, make sure you have vivado (tested with 2018.3) installed and in your path and run:

```
make -C ${BASE:?}/hardware/cva6 BOARD=kc705 fpga
``` 

## Software
The `software` folder contains several software components and assumes a working [RISC-V toolchain](https://github.com/riscv/riscv-gnu-toolchain/tree/13ed647cf231bf5875a3035aa9f2847af954c77e) installed on the system and available via the `RISCV` environment variable:

```
export RISCV=/path/to/riscv/toolchain
```

The patches in this repo already contain the instructions, but the `riscv-opcodes` repo should be run to populate the `env` subrepo in `riscv-tests`, and may also be used for further extensions. The riscv-openocd repo is not supplied in this repo, hence, set to `/dev/null` in the following:

```
make -C ${BASE:?}/software/riscv-opcodes clean
make -C ${BASE:?}/software/riscv-opcodes OPENOCD_H=/dev/null
```

The `riscv-isa-sim` repo contains an implementation of a functional model of SERVAS. For emulation performance reasons it does not implement an actual encryption using tweak but saves the tweak for each memory location and compares it during access.
Check the repos readme for additional information and run:

```
mkdir -p ${BASE:?}/software/riscv-isa-sim/build
cd software/riscv-isa-sim/build
../configure --prefix=$RISCV
make -j$(nproc)
make install
cd ${BASE:?}
```

The `riscv-tests` repo contains a number of baremetal tests that can be run in spike or on a SERVAS capable CPU. To build the tests:
```
cd ${BASE:?}/software/riscv-tests/
autoconf
./configure --prefix=$RISCV/../target
make isa -j$(nproc)
make install
cd ${BASE:?}
```

Individual tests can then be run like this:
```
spike --isa=rv64imz $BASE/software/riscv-tests/isa/rv64mz-p-basic_csr_access
```

For detailed instruction on the `ariane-sdk` consult its readme, the Makefile included in the readme has been adapted to also build `riscv-pk`. To quickly get a bootable linux image for spike and the hardware, run:

```
make -C ${BASE:?}/software/ariane-sdk bbl.bin
spike --log-level=0 -m256 --isa=rv64iafdcmz ${BASE:?}/software/ariane-sdk/bbl
```
