# How to build an iOS toolchain (on/for Linux)

In order to build for Apple platforms (iOS, MacOS, tvOS, etc), you need an environment that provides the relevant software developement and support tools. On MacOS, Apple provides this environment through its [XCode](https://developer.apple.com/xcode/whats-new/) app. On other platforms, however, we have to create said environment ourselves (see [Theos](https://github.com/theos/theos) and [Dragon](https://github.com/dragonbuild/dragon)).

Arguably the most integral item in one's developement environment is the toolchain -- a set of tools used to aid your R&D process -- which is ultimately responsible for the creation of your software product(s). With the growing list of developement tools, it can be daunting and laborious to find, build, and compile the necessary ones, especially when many of the necessary tools are made from separate entities with different documentation habits. This guide aims to concentrate some of that information to (hopefully) make the process of building and compiling an iOS toolchain clear and concise.

---

## The expected result

Our toolchain is primarily targeting iOS tweak developement and will contain the following: `llvm`, `clang`, `ldid`, `tapi`, `libtapi`, and `cctools-port`, though other tools (from LLVM or a third party) can be added as needed.

**Note:** Unless another target is specified explicitly prior to building (i.e., you want to [cross compile](https://llvm.org/docs/HowToCrossCompileLLVM.html)), the aforementioned tools will be built targeting the host system's [triple](https://clang.llvm.org/docs/CrossCompilation.html#target-triple) (e.g., `x86_64-unknown-linux-gnu` for my machine running [WSL](https://docs.microsoft.com/en-us/windows/wsl/about)).

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

### Create a centralized directory:

    mkdir -p $HOME/my-toolchain/

### Avoid possible conflicts:

If you've explicitly set `$LD_LIBRARY_PATH` and/or `$PATH` in your shell's profile, comment their declarations out (add "#" in front of the relevant lines) and restart your shell.

This is necessary because your other lib and bin paths may be prioritized over the default paths which contain the packages we just installed (depending on how you set the aforementioned variables). This prioritization has the potential to cause issues during compilation, so we're being proactive.

---

## 2. Clang+LLVM
### Important notes:

* If you have 8gb of ram or less, you’ll want to switch the linker in the cmake command below to either `gold` or `lld` (`-DLLVM_USE_LINKER=<linker>`) as they use less memory than the default linker `ld`.

* If you've switch the linker and still manage to encounter something similar to the following: “collect2: fatal error: ld terminated with signal 9 [Killed] compilation terminated” after running the make install command below, your jobs are still using too much memory. To resolve this, try running `make install` (i.e., a single job). If that still doesn't do it, take a look at the last two links in the 'Resouces' section below.

### The commands:

    git clone https://github.com/apple/llvm-project
    mkdir my-llvm-project && cd llvm-project && mkdir build && cd build
    cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS=clang -DLLVM_BUILD_LLVM_DYLIB=On -DLLVM_LINK_LLVM_DYLIB=On -DCMAKE_BUILD_TYPE=MinSizeRel -DCMAKE_INSTALL_PREFIX="$HOME/my-llvm-project/" ../llvm
    make -j"$(nproc --all)" install
    cd && mv $HOME/my-llvm-project/* $HOME/my-toolchain/

### Resources:

* https://github.com/apple/llvm-project#readme
* https://llvm.org/docs/CMake.html
* https://llvm.org/docs/BuildingADistribution.html
* https://stackoverflow.com/questions/48754619/what-are-cmake-build-type-debug-release-relwithdebinfo-and-minsizerel
* https://www.gnu.org/software/make/manual/html_node/Parallel.html
* https://stackoverflow.com/questions/65633304/not-able-to-build-llvm-from-its-source-code
* https://stackoverflow.com/questions/44807589/fatal-error-building-the-llvm-source-code-in-ubuntu

---

## 3. ldid

    git clone https://github.com/xerub/ldid
    cd ldid
    ./make.sh
    mv ldid2 $HOME/my-toolchain/bin/ldid && cd

### Resources:

* https://github.com/xerub/ldid#readme

---

## 4. cctools-port w/ tapi support

    git clone https://github.com/tpoechtrager/apple-libtapi
    mkdir cctools && cd apple-libtapi
    ./build.sh
    export INSTALLPREFIX="$HOME/cctools/" && ./install.sh && cd

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
