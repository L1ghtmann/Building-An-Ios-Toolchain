# How to build an iOS toolchain (on/for Linux)

In order to build for Apple platforms (iOS, MacOS, tvOS, etc), you need an environment that provides the relevant software development and support tools. On MacOS, Apple provides this environment through its [XCode](https://developer.apple.com/xcode/whats-new/) app. On other platforms, however, we have to create said environment ourselves (see [Theos](https://github.com/theos/theos) and [Dragon](https://github.com/dragonbuild/dragon)).

Arguably the most integral item in one's development environment is the toolchain -- a set of tools used to aid your R&D process -- which is ultimately responsible for the creation of your software product(s). With the growing list of development tools, it can be daunting and laborious to find, build, and compile the necessary ones, especially when many of the necessary tools are made from separate entities with different documentation habits. This guide aims to concentrate some of that information to (hopefully) make the process of building and compiling an iOS toolchain clear and concise.

---

## The expected result

Our toolchain is primarily targeting iOS tweak development and will contain the following: `llvm`, `clang`, `ldid`, `tapi`, `libtapi`, and `cctools-port`, though other tools (from LLVM or a third party) can be added as needed.

**Note:** Unless another target is specified explicitly prior to building (i.e., you want to [cross compile](https://llvm.org/docs/HowToCrossCompileLLVM.html)), the aforementioned tools will be built for use on the host system's [architecture](https://llvm.org/doxygen/classllvm_1_1Triple.html#a547abd13f7a3c063aa72c8192a868154) and [triple](https://clang.llvm.org/docs/CrossCompilation.html#target-triple) (e.g., `x86_64` and `x86_64-unknown-linux-gnu` for my machine running [WSL](https://docs.microsoft.com/en-us/windows/wsl/about)).

**Note 2:** The methods described below are what I've found works best. Yes, there are other ways to achieve the same or a similar end result. No, I will not be covering them here.

---

## 1. Preliminary Setup
### Install build dependencies:

    # General
    sudo apt-get install build-essential cmake coreutils git make

    # Clang+LLVM
    sudo apt-get install python3

    # cctools-port
    sudo apt-get install clang

**Note:** `build-essential` or the equivalent for your distro.

### Create a centralized directory:

    mkdir -p $HOME/my-toolchain/

### Avoid possible conflicts:

If you've explicitly set `$LD_LIBRARY_PATH` and/or `$PATH` in your shell's profile, comment their declarations out (add "#" in front of the relevant lines) and restart your shell.

This is necessary because your other lib and bin paths may be prioritized over the default paths which contain the packages we just installed (depending on how you set the aforementioned variables). This prioritization has the potential to cause issues during compilation, so we're being proactive.

---

## 2. Build Clang+LLVM

![infographic](https://i.stack.imgur.com/9xGDe.png) - [Stack Overflow](https://stackoverflow.com/a/49081640)

*"The LLVM Project is a collection of modular and reusable compiler and toolchain technologies."* - [LLVM Developer Group](https://llvm.org/)

### Important notes:

* If you have 8gb of ram or less, you’ll want to switch the linker in the cmake command below to either `gold` or `lld` (`-DLLVM_USE_LINKER=<linker>`) as they use less memory than the default linker `ld`.

* If you've switched the linker and manage to encounter something similar to the following: “collect2: fatal error: ld terminated with signal 9 [Killed] compilation terminated” after running the make install command below, your jobs are still using too much memory. To resolve this, try running `make install` (i.e., a single job). If that still doesn't do it, take a look at the last two links in the 'Resources' section below.

### The commands:

    git clone https://github.com/apple/llvm-project
    mkdir my-llvm-project && cd llvm-project && mkdir build && cd build
    cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS=clang -DLLVM_LINK_LLVM_DYLIB=On -DLLVM_ENABLE_WARNINGS=Off -DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64" -DCMAKE_BUILD_TYPE=MinSizeRel -DCMAKE_INSTALL_PREFIX="$HOME/my-llvm-project/" ../llvm
    make -j"$(nproc --all)" install
    cd && mv $HOME/my-llvm-project/* $HOME/my-toolchain/

### The flags explained:

`-G "Unix Makefiles"` tells CMake to use the "Unix Makefiles" generator. [From Wikipedia](https://en.wikipedia.org/wiki/CMake): *"CMake is not a build system but rather it generates another system's build files."*

`-DLLVM_ENABLE_PROJECTS=clang` specifies that we want to build `clang` alongside `llvm` as a sub-project.

`-DLLVM_LINK_LLVM_DYLIB=On` specifies that we want to build the `libLLVM` shared library and dynamically link it into all the tools we're about to build. This helps shrink the size of our compiler *significantly*.

`-DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64"` specifies that we want our compiler to support the x86, ARM, and AArch64 architecture families. By default, this flag is set to `all` which integrates support (i.e., builds a backend) for AArch64, AMDGPU, ARM, AVR, BPF, Hexagon, Lanai, Mips, MSP430, NVPTX, PowerPC, RISCV, Sparc, SystemZ, WebAssembly, X86, and XCore ([source](https://github.com/apple/llvm-project/blob/0c93705f060d7e0b8932d58c1fd6291dd6a3f5a9/llvm/CMakeLists.txt#L303)) most of which we will never use. In practice, we only need ARM, but having support for x86 and AArch64 won't hurt and they're the only other two architectures in that list that you're likely to encounter with modern Apple hardware.

`-DLLVM_ENABLE_WARNINGS=Off` specifies that we want to disable all compiler warnings. Trust me, they will get annoying. If something fails, it may be worth removing this flag to turn warnings back on.

`-DCMAKE_BUILD_TYPE=MinSizeRel` specifies that we want a Release build optimized for size, not speed.

`-DCMAKE_INSTALL_PREFIX="$HOME/my-llvm-project/"` specifies where we want our compiler to be installed once it's been built.

### Resources:

* https://llvm.org/docs/CMake.html
* https://github.com/apple/llvm-project#readme
* https://llvm.org/docs/BuildingADistribution.html
* https://groups.google.com/g/llvm-dev/c/-zRhXG8zC4Q/m/AuEhnjsAYj8J
* https://stackoverflow.com/questions/48754619/what-are-cmake-build-type-debug-release-relwithdebinfo-and-minsizerel
* https://www.gnu.org/software/make/manual/html_node/Parallel.html
* https://stackoverflow.com/questions/65633304/not-able-to-build-llvm-from-its-source-code
* https://stackoverflow.com/questions/44807589/fatal-error-building-the-llvm-source-code-in-ubuntu

---

## 3. Build ldid

*"ldid is a tool made by [Saurik](https://twitter.com/saurik) for modifying a binary's entitlements easily. ldid also generates SHA1 hashes for the binary signature, so the iPhone kernel executes the binary."* - [iPhoneDevWiki](https://iphonedev.wiki/index.php/Ldid)

### The commands:

    git clone https://github.com/xerub/ldid
    cd ldid
    ./make.sh
    mv ldid2 $HOME/my-toolchain/bin/ldid && cd

### Resources:

* https://github.com/xerub/ldid#readme

---

## 4. Build cctools-port w/ TAPI support

*"TAPI is a **T**ext-based **A**pplication **P**rogramming **I**nterface. It replaces the Mach-O Dynamic Library Stub files in Apple's SDKs to reduce the SDK size even further."* - [Apple](https://opensource.apple.com/source/tapi/tapi-1000.10.8/Readme.md)

*"[cctools is] a set of essential tools to support development on Mac OS X and Darwin. Conceptually similar to binutils on other platforms."* - [MacPorts](https://ports.macports.org/port/cctools/)


*cctools-port is a port of Apple's cctools and ld64 for Linux and \*BSD* - [Thomas Pöchtrager](https://github.com/tpoechtrager/cctools-port)

### The commands:

    # TAPI
    git clone https://github.com/tpoechtrager/apple-libtapi
    mkdir cctools && cd apple-libtapi
    ./build.sh
    export INSTALLPREFIX="$HOME/cctools/" && ./install.sh && cd

    # cctools-port
    git clone https://github.com/tpoechtrager/cctools-port
    cd cctools-port/cctools
    ./configure --prefix="$HOME/cctools/" --enable-tapi-support --with-libtapi="$HOME/cctools/"
    make -j"$(nproc --all)"
    make install
    cd && cp -a $HOME/cctools/* $HOME/my-toolchain/

### Resources:

* https://github.com/tpoechtrager/apple-libtapi#readme
* https://github.com/tpoechtrager/cctools-port#readme

---

## 5. Final Touches

Uncomment your previously commented out `$LD_LIBRARY_PATH` and/or `$PATH` declarations from your shell's profile and restart your shell.

Lastly, move the contents of `$HOME/my-toolchain/` to the desired location.

---

And, voilà, you have an iOS toolchain!

~ Lightmann
