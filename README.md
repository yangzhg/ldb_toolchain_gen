# Linux Distribution Based Toolchain Generator

## Introduction

LDB toolchain generator help generate toolchains derived from well-known linux distributions (LDB toolchain). LDB toolchain contains binaries from well-known linux distributions, which are heavily tested and optimized. It avoids the docker dependency which usually complicate the developing workflow. Projects adopting LDB toolchain will likely generate byte-identical binaries. It benefits testing and debugging due to 100% consistency. It also lowers the bar of contributing code, as large C/C++ projects are usually daunting to build compared to Java, Golang, Rust, etc.

There are projects like Crosstool-NG and Gentoo-Prefix which aim at bootstraping toolchains from scratch. They are for advanced use cases and take a tremendous amount of time to bootstrap. It's also hard to control the quality of self-compiled toolchains.

## Tools used by LDB toolchain generator

1. https://github.com/intoli/exodus
2. https://github.com/NixOS/patchelf

[Exodus](https://github.com/intoli/exodus) generates relocate Linux ELF binaries with wrappers and dependencies. Dependencies are recursively collected. Wrappers are usually good enough for common tools. However, toolchains like GCC and Clang relies heavily on `/proc/self` to re-exec the driver program multiple times, which will not work for `ld-linux` wrappers. As a result, we use [patchelf](https://github.com/NixOS/patchelf) to modify `PT_INTERP` and `RPATH`. Though `RPATH` can be relative to the binary's location, it's not possible to make a relocatable a.out that can tolerate changes to the absolute path of `ld-linux`. Thus, LDB toolchain generator requires user to specify a **prefix**.

## How to use

The main branch illustrate how the generator assembles gcc-11 and clang-13 from ubuntu-18.04.

To actually generate the toolchain, the following steps can be used:

```
git clone https://github.com/amosbird/ldb_toolchain_gen

cd ldb_toolchain_gen

docker build . -t <your_toolchain_generator_tag>

docker run --rm -v </path/to/store/toolchain>:/data <your_toolchain_generator_tag> <your_toolchain_prefix>

# Concrete Example

git clone https://github.com/amosbird/ldb_toolchain_gen

cd ldb_toolchain_gen

# You may need http proxy to overcome network errors during docker build
docker build . -t amosbird/some_toolchain_gen
# docker build --network host --build-arg http_proxy=http://127.0.0.1:10000 --build-arg https_proxy=http://127.0.0.1:10000 . -t amosbird/some_toolchain_gen

mkdir /tmp/some_toolchain_gen

# You may need root privilege
docker run --rm -v /tmp/some_toolchain_gen:/data amosbird/some_toolchain_gen /tmp/some_toolchain

# You may need root privilege
mv /tmp/some_toolchain_gen/toolchain /tmp/some_toolchain

export PATH=/tmp/some_toolchain/bin:$PATH

gcc -v

clang -v
```

In the end, `/path/to/store/toolchain` will have a `toolchain` directory and a `toolchain.tgz` tarball. Make sure the toolchain is relocated to `<your_toolchain_prefix>` before use.

The main branch is used to build [ClickHouse](https://github.com/ClickHouse/ClickHouse). There are also different branches targeting specific projects. For instance, branch doris-ubuntu-18.04-x64 is mainly used for building [Doris](https://github.com/apache/incubator-doris).

## How to avoid GLIBC incompatibility

LDB toolchain might use newer libc which in turn will generate binaries that cannot be run on old host, even the same host for compilation. Users will encounter errors like `version GLIBC_2.27 not found`. A practical way of resolving this issue is described at http://www.lightofdawn.org/wiki/wiki.cgi/NewAppsOnOldGlibc .

## Pre-built images

In case building the generator image is impossible, currently two images are provided: `amosbird/ldb_toolchain:ubuntu_18_04_x64` and `amosbird/ldb_toolchain:doris_ubuntu_18_04_x64`. Note, the images are bloated and not optimized at all.

```
mkdir /tmp/some_toolchain_gen

# You may need root privilege
docker run --rm -v /tmp/some_toolchain_gen:/data amosbird/ldb_toolchain:ubuntu_18_04_x64 /tmp/some_toolchain

# You may need root privilege
mv /tmp/some_toolchain_gen/toolchain /tmp/some_toolchain

export PATH=/tmp/some_toolchain/bin:$PATH

gcc -v

clang -v
```

TODO: Provide dedicated service to generate toolchains with given prefix and publish the toolchain.tgz.
