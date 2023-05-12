# GCC-Cross-Compiler-Aarch64
## Environment
```
Ubuntu 22.04.1
```

## Required Packages
Starting with a clean Ubuntu system, you have to install a few packages:
```
sudo apt-get install g++ make gawk wget
```

Everything else will built from source, so that we have to download the following source packages.
```
wget http://ftpmirror.gnu.org/binutils/binutils-2.24.tar.gz
wget http://ftpmirror.gnu.org/gcc/gcc-8.3.0/gcc-8.3.0.tar.gz
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.17.2.tar.xz
wget http://ftpmirror.gnu.org/glibc/glibc-2.20.tar.xz
wget http://ftpmirror.gnu.org/mpfr/mpfr-3.1.2.tar.xz
wget http://ftpmirror.gnu.org/gmp/gmp-6.0.0a.tar.xz
wget http://ftpmirror.gnu.org/mpc/mpc-1.0.2.tar.gz
```

## Build Steps

## 0. Preprocess
Extract all the source packages
```
for f in *.tar*; do tar xf $f; done
rm *.tar.*
```

Create symbolic links from the GCC directory to some of the others directories.
```
cd gcc-8.3.0
ln -s ../mpfr-3.1.2 mpfr
ln -s ../gmp-6.0.0 gmp
ln -s ../mpc-1.0.2 mpc
cd ..
```

Choose an installation directory, and make sure you have write permission to it. In that steps that follow, I will install the builded toolchain to `/opt/cross`.
```
sudo su
sudo mkdir -p /opt/cross
```

Throughout the entire build process, make sure the installation's `bin` subdirectory is in your `PATH` environment variable.
```
export PATH=/opt/cross/bin:$PATH
```

## 1. Binutils
This step builds and installs the cross-assembler, cross-linker, and other tools.
```
mkdir build-binutils
cd build-binutils
../binutils-2.24/configure --prefix=/opt/cross --target=aarch64-linux --disable-multilib --disable-werror
make -j$(nproc)
make install
cd ..
```
* Binutils's `configure` script will recognize that this target is different from the machine we're building on, and configure a cross-assembler and cross-linker as a result. The tools will be installed to `/opt/cross/bin`, their names prefixed by target system type (`aarch64-linux-`).
* `target=aarch64-linux` means that target system type.
* `--disable-multilib` means that we only want our Binutils installation to work with programs and libraries using the AArch64 instruction set, and not any related instruction sets such as AArch32.
* `--disable-werror` helps us to avoid error. (Meaning UNKNOW)

## 2. Linux Kernel Headers
This step installs the Linux kernel header files to `/opt/cross/aarch64-linux/include`, which will ultimately allow programs built using our builded toolchain to make system calls to AArch64 kernel in the target environment.
```
cd linux-3.17.2
make ARCH=arm64 INSTALL_HDR_PATH=/opt/cross/aarch64-linux headers_install
cd ..
```
* Because the Linux kernel is a different open-source project from the others, it has a different way of identifying the target CPU architecture by `ARCH=arm64`

## 3. C/C++ Compilers
This step will build GCC's C and C++ cross-compiler only, and install them to `/opt/cross/bin`. It wouldn't invoke those compilers to build any libraries yet.
```
mkdir -p build-gcc
cd build-gcc
../gcc-8.3.0/configure --prefix=/opt/cross --target=aarch64-linux --enable-languages=c,c++ --disable-multilib
make -j$(nproc) all-gcc
make install-gcc
cd ..
```
* `--enable-languages=c,c++` prevents other compilers in the GCC suite, such as Fortran, Go or Java, from being built.

## 4. Standard C Library Headers and Startup Files
In this step, we install Glibc's standard C library headers to `/opt/cross/aarch64-linux/include`. We also use the C compiler built in step 3 to compile the library's startup files and install them to `/opt/cross/aarch64-linux/lib`. Finally, we create a couple of dummy files, `libc.so` and `stubs.h`, which are expected in step 5, but which will be replaced in step 6.
```
mkdir -p build-glibc
cd build-glibc
../glibc-2.20/configure --prefix=/opt/cross/aarch64-linux --build=$MACHTYPE --host=aarch64-linux --target=aarch64-linux --with-headers=/opt/cross/aarch64-linux/include --disable-multilib libc_cv_forced_unwind=yes
make install-bootstrap-headers=yes install-headers
make -j$(nproc) csu/subdir_lib
install csu/crt1.o csu/crti.o csu/crtn.o /opt/cross/aarch64-linux/lib
aarch64-linux-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o /opt/cross/aarch64-linux/lib/libc.so
touch /opt/cross/aarch64-linux/include/gnu/stubs.h
cd ..
```
* `$MACHTYPE` is a pre-defined environment variable which describes the machine running the build script. This is needed because in step 6, the build script will compile some addtional tools which run as part of the build process itself.
* Install the C library's startup files, `crt1.o`, `crti.o` and `crtn.o`, to the installation directory manually.

## 5. Compiler Support Library
This step uses the cross-compilers built in step 3 to build the compiler support library. The compiler support library contains some C++ exception handling boilerplate code, among other things. This library depends on the startup files installed in step 4. The library itself is needed in step 6.
```
cd build-gcc
make -j$(nproc) all-target-libgcc
make install-target-libgcc
cd ..
```
* Two static libraries, `libgcc.a` and `libgcc_eh.a`, are installed to `/opt/cross/lib/gcc/aarch64-linux/4.9.2/`.
* A shared library, `libgcc_s.so` is installed to `/opt/cross/aarch64-linux/lib64`.

## 6. Standard C Library
In this step, we finish off the Glibc package, which builds the standard C library and installs its files to `/opt/cross/aarch64-linux/lib`. The static library is named `libc.a` and the shared library is `libc.so`.
```
cd build-glibc
make -j$(nproc)
make install
cd ..
```

## 7. Standard C++ Library
Finally, we finish off the GCC package, which builds the standard C++ library and installs it to `/opt/cross/aarch64-linux/lib64/`. It depends on the C library build in step 6. The resulting static library is named `libstdc++.a` and the shared library is `libstdc++.so`.
```
cd build-gcc
make -j$(nproc)
make install
cd ..
```

## Testing the Cross-Compiler
If everything build successfully, let us check our cross-compiler for a dial tone:
```
aarch64-linux-g++ -v
Using built-in specs.
COLLECT_GCC=aarch64-linux-g++
COLLECT_LTO_WRAPPER=/opt/cross/libexec/gcc/aarch64-linux/8.3.0/lto-wrapper
Target: aarch64-linux
Configured with: ../gcc-8.3.0/configure --prefix=/opt/cross --target=aarch64-linux --enable-languages=c,c++ --disable-multilib
Thread model: posix
gcc version 8.3.0 (GCC)
```

We can compile the C++14 program `hello.c++`, then disassemble it:
```
aarch64-linux-g++ -std=c++14 hello.cpp
aarch64-linux-objdump -d a.out
```

## References
* [How to Build a GCC Cross-Compiler](https://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/)
* [GCC Cross-Compiler](https://wiki.osdev.org/GCC_Cross-Compiler#The_Build)
