[![Build Status](https://travis-ci.org/herumi/mcl.png)](https://travis-ci.org/herumi/mcl)

# mcl

A generic and fast pairing-based cryptography library.

# Abstract

mcl is a library for pairing-based cryptography.
The current version supports the optimal Ate pairing over BN curves.

# Support architecture

* x86-64 Windows + Visual Studio
* x86, x86-64 Linux + gcc/clang
* ARM Linux
* ARM64 Linux
* (maybe any platform to be supported by LLVM)

# Installation Requirements

* [GMP](https://gmplib.org/)
```
apt install libgmp-dev
```

Create a working directory (e.g., work) and clone the following repositories.
```
mkdir work
cd work
git clone git://github.com/herumi/mcl
git clone git://github.com/herumi/cybozulib
git clone git://github.com/herumi/xbyak ; for only x86/x64
git clone git://github.com/herumi/cybozulib_ext ; for only Windows
```
* Cybozulib_ext is a prerequisite for running OpenSSL and GMP on VC (Visual C++).

# Build and test on x86-64 Linux, macOS, ARM and ARM64 Linux
To make lib/libmcl.a and test it:
```
cod work/mcl
make test
```
To benchmark a pairing:
```
bin/bn_test.exe
```
To make sample programs:
```
make sample
```

if you want to change compiler options for optimization, then set `CFLAGS_OPT_USER`.
```
make CLFAGS_OPT_USER="-O2"
```

## Build for 32-bit Linux
Build openssl and gmp for 32-bit mode and install `<lib32>`
```
make ARCH=x86 CFLAGS_USER="-I <lib32>/include" LDFLAGS_USER="-L <lib32>/lib -Wl,-rpath,<lib32>/lib"
```

## Build for 64-bit Windows
1) make library
```
mklib.bat
```
2) make exe binary of sample\pairing.cpp
```
mk sample\pairing.cpp
bin/bn_test.exe
```

open mcl.sln and build or if you have msbuild.exe
```
msbuild /p:Configuration=Release
```

## Build with cmake
```
mkdir build
cd build
cmake ..
make
```
### SELinux
mcl uses Xbyak JIT engine if it is available on x64 architecture,
otherwise mcl uses a little slower functions generated by LLVM.
The default mode enables SELinux security policy on CentOS, then JIT is disabled.
```
% sudo setenforce 1
% getenforce
Enforcing
% bin/bn_test.exe
JIT 0
pairing   1.496Mclk
finalExp 581.081Kclk

% sudo setenforce 0
% getenforce
Permissive
% bin/bn_test.exe
JIT 1
pairing   1.394Mclk
finalExp 546.259Kclk
```

# Libraries

* libmcl.a ; static C++ library of mcl
* libmcl_dy.so ; shared C++ library of mcl
* libbn256.a ; static C library for `mcl/bn256f.h`
* libbn256_dy.so ; shared C library

# How to initialize pairing library
Call `mcl::bn256::bn256init` before calling any operations.
```
#include <mcl/bn256.hpp>
mcl::bn::CurveParam cp = mcl::bn::CurveFp254BNb; // or mcl::bn::CurveSNARK1
mcl::bn256::bn256init(cp);
mcl::bn256::G1 P(...);
mcl::bn256::G2 Q(...);
mcl::bn256::Fp12 e;
mcl::bn256::BN::pairing(e, P, Q);
```
1. (CurveFp254BNb) a BN curve over the 254-bit prime p = 36z^4 + 36z^3 + 24z^2 + 6z + 1 where z = -(2^62 + 2^55 + 1).
2. (CurveSNARK1) a BN curve over a 254-bit prime p such that n := p + 1 - t has high 2-adicity.

See [test/bn_test.cpp](https://github.com/herumi/mcl/blob/master/test/bn_test.cpp).

## Default constructor of Fp, Ec, etc.
A default constructor does not initialize the instance.
Set a valid value before reffering it.

## String format of G1 and G2
G1 and G2 have three elements of Fp (x, y, z) for Jacobi coordinate.
normalize() method normalizes it to affine coordinate (x, y, 1) or (0, 0, 0).
G1::setCompressedExpression(bool) sets whether uncompressed(false) or compressed(true) format.

getStr() method gets

* `0` ; infinity
* `1 <x> <y>` ; not compressed format
* `2 <x>` ; compressed format for even y
* `3 <x>` ; compressed format for odd y

## Verify an element in G2
`G2::isValid()` checks that the element is in the curve of G2 and the order of it is r.
`G2::set()`, `G2::setStr` and `operator<<` also check the order.
If you check it out of the library, then you can stop the verification by calling `G2::setOrder(0)`.

# Benchmark

A benchmark of a BN curve CurveFp254BNb(2016/12/25).

* x64, x86 ; Inte Core i7-6700 3.4GHz(Skylake) upto 4GHz on Ubuntu 16.04.
    * `sudo cpufreq-set -g performance`
* arm ; 900MHz quad-core ARM Cortex-A7 on Raspberry Pi2, Linux 4.4.11-v7+
* arm64 ; 1.2GHz ARM Cortex-A53 [HiKey](http://www.96boards.org/product/hikey/)

software                                                 |   x64|  x86| arm|arm64(msec)
---------------------------------------------------------|------|-----|----|-----
[ate-pairing](https://github.com/herumi/ate-pairing)     | 0.21 |   - |  - |    -
mcl                                                      | 0.31 | 1.6 |22.6|  4.0
[TEPLA](http://www.cipher.risk.tsukuba.ac.jp/tepla/)     | 1.76 | 3.7 | 37 | 17.9
[RELIC](https://github.com/relic-toolkit/relic) PRIME=254| 0.30 | 3.5 | 36 |    -
[MIRACL](https://github.com/miracl/MIRACL) ake12bnx      | 4.2  |   - | 78 |    -
[NEONabe](http://sandia.cs.cinvestav.mx/Site/NEONabe)    |   -  |   - | 16 |    -

* compile option for RELIC
```
cmake -DARITH=x64-asm-254 -DFP_PRIME=254 -DFPX_METHD="INTEG;INTEG;LAZYR" -DPP_METHD="LAZYR;OATEP"
```
# 384-bit curve (experimental)
see `test/bn384_test.cpp`
Benchmark on Skylake(3.4GHz)

```
# mcl::bn::CurveFp382_1 ; -(2^94 + 2^76 + 2^72 + 1)
pairing   3.163Mclk  ; 0.93msec

# mcl::bn::CurveFp382_2 ; -(2^94 + 2^78 + 2^67 + 2^64 + 2^48 + 1)
pairing   3.261Mclk  ; 0.96msec
```

# How to make asm files (optional)
The asm files generated by this way are already put in `src/asm`, then it is not necessary to do this.

Install [LLVM](http://llvm.org/).
```
make MCL_USE_LLVM=1 LLVM_VER=<llvm-version> UPDATE_ASM=1
```
For example, specify `-3.8` for `<llvm-version>` if `opt-3.8` and `llc-3.8` are installed.

If you want to use Fp with 1024-bit prime on x86-64, then
```
make MCL_USE_LLVM=1 LLVM_VER=<llvm-version> UPDATE_ASM=1 MCL_MAX_BIT_SIZE=1024
```

# Java API
See [java.md](https://github.com/herumi/mcl/blob/master/java/java.md)

# License

modified new BSD License
http://opensource.org/licenses/BSD-3-Clause

The original source of the followings are https://github.com/aistcrypt/Lifted-ElGamal .
These files are licensed by BSD-3-Clause and are used for only tests.

```
include/mcl/elgamal.hpp
include/mcl/window_method.hpp
test/elgamal_test.cpp
test/window_method_test.cpp
sample/vote.cpp
```
This library contains [mie](https://github.com/herumi/mie/) and [Lifted-ElGamal](https://github.com/aistcrypt/Lifted-ElGamal/).

# References
* [ate-pairing](https://github.com/herumi/ate-pairing/)
* [_Faster Explicit Formulas for Computing Pairings over Ordinary Curves_](http://dx.doi.org/10.1007/978-3-642-20465-4_5),
 D.F. Aranha, K. Karabina, P. Longa, C.H. Gebotys, J. Lopez,
 EUROCRYPTO 2011, ([preprint](http://eprint.iacr.org/2010/526))
* [_High-Speed Software Implementation of the Optimal Ate Pairing over Barreto-Naehrig Curves_](http://dx.doi.org/10.1007/978-3-642-17455-1_2),
   Jean-Luc Beuchat, Jorge Enrique González Díaz, Shigeo Mitsunari, Eiji Okamoto, Francisco Rodríguez-Henríquez, Tadanori Teruya,
  Pairing 2010, ([preprint](http://eprint.iacr.org/2010/354))
* [_Faster hashing to G2_](http://dx.doi.org/10.1007/978-3-642-28496-0_25),Laura Fuentes-Castañeda,  Edward Knapp,  Francisco Rodríguez-Henríquez,
  SAC 2011, ([preprint](https://eprint.iacr.org/2008/530))
* [_Skew Frobenius Map and Efficient Scalar Multiplication for Pairing–Based Cryptography_](https://www.researchgate.net/publication/221282560_Skew_Frobenius_Map_and_Efficient_Scalar_Multiplication_for_Pairing-Based_Cryptography),
Y. Sakemi, Y. Nogami, K. Okeya, Y. Morikawa, CANS 2008.

# Author

光成滋生 MITSUNARI Shigeo(herumi@nifty.com)
