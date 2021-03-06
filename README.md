
This repository implements a mechanism to perform bpf program verification
in user space. It further utilizes llvm sanitizer and fuzzer framework to
extend error detection coverage.

## Motivation

The motivation of this project is to test verifier in userspace so that
we can take advantage of llvm's sanitizer and fuzzer framework. One bug
has been discovered because of this effort:
http://permalink.gmane.org/gmane.linux.network/376864

## Directory Overview

```
  - bld
  - config
  - src
    - helper
    - test
      - linux-samples-bpf
      - fuzzer
```

The bld directory is used to build and run test programs.
The config directory holds the recommended linux config file
The src/helper contains helper files for kernel and 
user hooks. The src/test/linux-samples-bpf contains 
related test verifier files from linux/samples/bpf/ and
src/test/fuzzer contains a hook with llvm fuzzer framework.

## Prerequisite

A linux source tree is needed. The source tree is used to pre-process kernel files.
Note that kernel headers will need to be generated at default <kernel_root>/usr/include
directory. The default config does not have all necessary BPF options enabled,
and will make verifier tests fail.
you can try to use the one at the config directory instead of default
linux:arch/x86/configs/x86_64_defconfig.

```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git
# change defconfig and apply necessary patch as decribed below
cd net-next
make defconfig
make headers_install
```

For above linux tree, apply the following patch so that llvm can cope with linux
inline assembly:

```
yhs@ubuntu:~/work/fuzzer/net-next$ git diff
diff --git a/Makefile b/Makefile
index c361593..cacbe0f 100644
--- a/Makefile
+++ b/Makefile
@@ -686,6 +686,8 @@ KBUILD_CFLAGS += $(call cc-disable-warning, tautological-compare)
 # See modpost pattern 2
 KBUILD_CFLAGS += $(call cc-option, -mno-global-merge,)
 KBUILD_CFLAGS += $(call cc-option, -fcatch-undefined-behavior)
+# no integrated assembler so not checking inlining assembly format
+KBUILD_CFLAGS += $(call cc-option, -no-integrated-as)
 else
 
 # This warning generated too much noise in a regular build.
yhs@ubuntu:~/work/fuzzer/net-next$ 
```

A llvm/clang compiler with compiler-rt is needed. The compiler-rt is necessary
for llvm sanitizer support.

```bash
sudo apt-get -y install bison build-essential cmake flex git libedit-dev python zlib1g-dev
git clone http://llvm.org/git/llvm.git
cd llvm/tools; git clone http://llvm.org/git/clang.git
cd ../projects; git clone http://llvm.org/git/compiler-rt.git
cd ..; mkdir -p build/install; cd build
cmake -G "Unix Makefiles" -DLLVM_TARGETS_TO_BUILD="BPF;X86" -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$PWD/install ..
make -j4
make install
export PATH=$PWD/install/bin:$PATH
```

## Build and Run

```bash
export KERNEL_TREE_ROOT=<kernel_tree_root>
cd bld
make setup
make all
```

The "setup" target creates common symbolic links and download
and build llvm fuzzer. It only needs to run once.

The "all" target builds two binaries, `test_verifier` and `test_fuzzer`.
`test_verifier` is essentially `linux/samples/bpf/test_verifier.c` with
slight modification to adapt to the new test framework, with
sanitizer support. `test_fuzzer` uses llvm fuzzer framework
for testing.

`test_verifier` can also generate initial test cases for `test_fuzzer`.
These test cases will make fuzzer more effective in generating relevent
new test cases.
```bash
# generate initial test cases for fuzzer, put them into ./corpus directory
./test_verifier -g ./corpus
./test_fuzzer -max_len=1024 ./corpus
```

To report memory leaks, the `test_fuzzer` needs to terminate normally.
This can be done by limiting the number of runs, e.g.,
```bash
./test_fuzzer -max_len=1024 -runs=1000000 ./corpus
```

The following command can be used to find the number of coverages:
```bash
# the actual number of coverages should be the number of lines minus one
# to account for function declaration.
objdump -D test_fuzzer | grep '<__sanitizer_cov>' | wc
```

With this information, you can compare to `test_fuzzer` output to
see what is the coverage. For example, if the total number of
edge coverages is 1106, and you have the following from `test_fuzzer`
output:
```
......
#3466257868	NEW    cov: 884 bits: 3749 units: 936 exec/s: 38408 L: 47
#3486994800	NEW    cov: 884 bits: 3750 units: 937 exec/s: 38396 L: 64
#3491234878	NEW    cov: 886 bits: 3752 units: 938 exec/s: 38394 L: 58
#3495108828	NEW    cov: 886 bits: 3754 units: 939 exec/s: 38391 L: 58
```
You will know 886 out of 1106 edges are covered so far.
