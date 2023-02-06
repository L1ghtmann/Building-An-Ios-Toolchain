# How to build an iOS toolchain (on/for Linux)

In order to build for Apple platforms (iOS, MacOS, tvOS, etc), you need an environment that provides the relevant software development and support tools. On MacOS, Apple provides this environment through its [XCode](https://developer.apple.com/xcode/whats-new/) app. On other platforms, however, we have to create said environment ourselves (see [Theos](https://github.com/theos/theos) and [Dragon](https://github.com/dragonbuild/dragon)).

Arguably the most integral item in one's development environment is the toolchain -- a set of tools used to aid your R&D process -- which is ultimately responsible for the creation of your software product(s). With the growing list of development tools, it can be daunting and laborious to find, build, and compile the necessary ones, especially when many of the necessary tools are made from separate entities with different documentation habits. This guide aims to concentrate some of that information to (hopefully) make the process of building and compiling an iOS toolchain clear and concise.

---

## The expected result

Our toolchain is primarily targeting iOS tweak development and will contain the following: `llvm`, `clang`, `ldid`, `(lib)tapi`, and `cctools-port`, though other projects (from LLVM or a third party) can be added as desired.

**Note:** Unless another target is specified explicitly prior to building (i.e., you want to [cross compile](https://llvm.org/docs/HowToCrossCompileLLVM.html)), the aforementioned tools will be built for use on the host system's [architecture](https://llvm.org/doxygen/classllvm_1_1Triple.html#a547abd13f7a3c063aa72c8192a868154) and [triple](https://clang.llvm.org/docs/CrossCompilation.html#target-triple) (e.g., `x86_64` and `x86_64-unknown-linux-gnu` for my machine running [WSL](https://docs.microsoft.com/en-us/windows/wsl/about)).

**Note 2:** The methods described below are what I've found works best. Yes, there are other ways to achieve the same or a similar end result. No, I will not be covering them here.

---

## 1. Preliminary Setup
### Install necessary dependencies:

	sudo apt install build-essential \
		autoconf \
		automake \
		cmake \
		coreutils \
		git \
		libssl-dev \
		libtool \
		make \
		pkg-config \
		python3

**Note:** These dependencies are for Debian-based distros. For other distros, you'll need to determine the equivalent packages.

### Create a centralized directory:

	mkdir -p $HOME/my-toolchain/

### Avoid possible conflicts:

If you've explicitly set `$LD_LIBRARY_PATH` and/or `$PATH` in your shell's profile, comment their declarations out (add "#" in front of the relevant lines) and restart your shell.

This is necessary because your other lib and bin paths may be prioritized over the default paths which contain the tools we'll be using (depending on how you set the aforementioned variables). This prioritization has the potential to cause issues during compilation, so we're being proactive.

---

## 2. Build Clang+LLVM

![infographic](https://i.stack.imgur.com/9xGDe.png) - [Stack Overflow](https://stackoverflow.com/a/49081640)

*"The LLVM Project is a collection of modular and reusable compiler and toolchain technologies."* - [LLVM Developer Group](https://llvm.org/)

### Important notes:

* If you have 8gb of ram or less, you’ll want to switch the linker in the cmake command below to either `gold` or `lld` (`-DLLVM_USE_LINKER=<linker>`) as they use less memory than the default linker `ld`.

* If you've switched the linker and manage to encounter something similar to the following:

	> collect2: fatal error: ld terminated with signal 9 [Killed] compilation terminated.

	after running the make install command below, your jobs are still using too much memory. To resolve this, try running `make install` (i.e., a single job).

	- If that still doesn't do it, take a look at the last two links in the [resources](#clangllvm-resources) section below.

### The commands:

	git clone --depth 1 https://github.com/apple/llvm-project
	cd llvm-project && mkdir build && cd build
	cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS=clang \
		-DLLVM_LINK_LLVM_DYLIB=ON \
		-DLLVM_ENABLE_LIBXML2=OFF \
		-DLLVM_ENABLE_ZLIB=OFF \
		-DLLVM_ENABLE_Z3_SOLVER=OFF \
		-DLLVM_ENABLE_BINDINGS=OFF \
		-DLLVM_ENABLE_WARNINGS=OFF \
		-DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64" \
		-DLLVM_INCLUDE_TESTS=OFF \
		-DCLANG_INCLUDE_TESTS=OFF \
		-DCMAKE_BUILD_TYPE=MinSizeRel \
		-DCMAKE_INSTALL_PREFIX="$HOME/my-toolchain/" \
		../llvm
	make -j$(nproc --all) install

### Notes:

`-G "Unix Makefiles"` tells CMake to use the Makefile generator. [From Wikipedia](https://en.wikipedia.org/wiki/CMake): *"CMake is not a build system but rather it generates another system's build files."*

`-DLLVM_ENABLE_PROJECTS=clang` specifies that we want to build `clang` alongside `llvm` as a sub-project.

`-DLLVM_LINK_LLVM_DYLIB=ON` specifies that we want to build the `libLLVM` shared library and dynamically link it into all the tools we're about to build. This helps shrink the size of our compiler *significantly*.

`-DLLVM_ENABLE_LIBXML2=OFF` specifies that we want to disable the reliance on libxml2 (removes a possible dependency).

`-DLLVM_ENABLE_ZLIB=OFF` specifies that we want to disable the reliance on zlib (removes a possible dependency).

`-DLLVM_ENABLE_Z3_SOLVER=OFF` specifies that we want to disable the Z3 constraint solver provided by the Clang static analyzer (thus removing yet another possible dependency).

`-DLLVM_ENABLE_BINDINGS=OFF` specifies that we don't want to support bindings for other languages (e.g., go and OCaml).

`-DLLVM_ENABLE_WARNINGS=OFF` specifies that we want to disable all compiler warnings. Trust me, they will get annoying. If something fails, it may be worth removing this flag to turn warnings back on.

`-DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64"` specifies that we want our compiler to support the x86, ARM, and AArch64 architecture families. By default, this flag is set to `all` which integrates support (i.e., builds a backend) for AArch64, AMDGPU, ARM, AVR, BPF, Hexagon, Lanai, Mips, MSP430, NVPTX, PowerPC, RISCV, Sparc, SystemZ, WebAssembly, X86, and XCore ([source](https://github.com/apple/llvm-project/blob/0c93705f060d7e0b8932d58c1fd6291dd6a3f5a9/llvm/CMakeLists.txt#L303)) most of which we'll never use. In practice, we only need AArch64 and ARM (64-bit and 32-bit arm respectively), but having support for the x86 family won't hurt and it's the only other architecture set in that list that you're likely to target (for other projects, etc).

`-DLLVM_INCLUDE_TESTS=OFF` specifies that we want to skip generating build targets for LLVM's unit tests.

`-DCLANG_INCLUDE_TESTS=OFF` specifies that we want to skip generating build targets for clang's unit tests.

`-DCMAKE_BUILD_TYPE=MinSizeRel` specifies that we want a release build optimized for size, not speed.

`-DCMAKE_INSTALL_PREFIX="$HOME/my-toolchain/"` specifies where we want our compiler to be installed once it's been built.

### Clang+LLVM resources:

* https://llvm.org/docs/CMake.html
* https://github.com/apple/llvm-project#readme
* https://llvm.org/docs/BuildingADistribution.html
* https://groups.google.com/g/llvm-dev/c/-zRhXG8zC4Q/m/AuEhnjsAYj8J
* https://stackoverflow.com/questions/48754619/what-are-cmake-build-type-debug-release-relwithdebinfo-and-minsizerel
* https://www.gnu.org/software/make/manual/html_node/Parallel.html
* https://stackoverflow.com/questions/65633304/not-able-to-build-llvm-from-its-source-code
* https://stackoverflow.com/questions/44807589/fatal-error-building-the-llvm-source-code-in-ubuntu

---

## 3. Build libplist

*libplist is a "library to handle Apple Property List format in binary or XML."* - [libimobiledevice](https://github.com/libimobiledevice)

### The commands:

	git clone --depth=1 https://github.com/libimobiledevice/libplist
	mkdir -p $HOME/libplist && cd libplist
	./autogen.sh --prefix="$HOME/libplist" --without-cython
	make -j$(nproc --all) install

### Notes:

- We build libplist from source in order to get an up-to-date version, as the release builds are typically outdated, and in order to get a correctly named static archive.
  - This archive will be used when linking ldid (see below).

### libplist resources:

* https://github.com/libimobiledevice/libplist
* https://libimobiledevice.org/

---

## 4. Build ldid

*"ldid is a tool made by [Saurik](https://twitter.com/saurik) for modifying a binary's entitlements easily. ldid also generates SHA1 hashes for the binary signature, so the iPhone kernel executes the binary."* - [iPhoneDevWiki](https://iphonedev.wiki/index.php/Ldid)

### The commands:

	git clone --depth=1 https://github.com/ProcursusTeam/ldid
	cd ldid
	make -j$(nproc --all) DESTDIR="$HOME/my-toolchain/" \
		PREFIX="" \
		LIBCRYPTO_LIBS="-l:libcrypto.a -lpthread -ldl" \
		LIBPLIST_INCLUDES="-I$HOME/libplist/include" \
		LIBPLIST_LIBS="$HOME/libplist/lib/libplist-2.0.a" \
	install

### Notes:

- We specify that we want to statically link libcrypto and libplist as the versions can vary significantly between systems.
  - Furthermore, doing so will prevent libplist from having to be installed on the host system as a dependency (hence the additional include path passed).
- The additional dynamic links to pthread and dl are required on some older systems (e.g., Ubuntu 18.04).

### ldid resources:

* https://github.com/ProcursusTeam/ldid
* https://git.saurik.com/ldid.git

---

## 5. Build lib(tapi)

*"TAPI is a **T**ext-based **A**pplication **P**rogramming **I**nterface. It replaces the Mach-O Dynamic Library Stub files in Apple's SDKs to reduce the SDK size even further."* - [Apple](https://opensource.apple.com/source/tapi/tapi-1000.10.8/Readme.md)

### The commands:

	git clone --depth=1 https://github.com/tpoechtrager/apple-libtapi
	mkdir -p $HOME/cctools && cd apple-libtapi
	mkdir build-tblgens && cd build-tblgens
	cmake -G "Unix Makefiles" -DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64" \
		-DLLVM_INCLUDE_TESTS=OFF \
		-DLLVM_ENABLE_WARNINGS=OFF \
		-DCLANG_INCLUDE_TESTS=OFF \
		-DCMAKE_BUILD_TYPE=Release \
		../src/llvm
	make -j$(nproc --all) llvm-tblgen clang-tblgen

	cd ../ && mkdir build-tapi && cd build-tapi
	cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS="libtapi" \
		-DLLVM_INCLUDE_TESTS=OFF \
		-DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64" \
		-DLLVM_ENABLE_WARNINGS=OFF \
		-DTAPI_FULL_VERSION="$(cat $PWD/../VERSION.txt | grep "tapi" | grep -o '[[:digit:]].*')" \
		-DLLVM_TABLEGEN="$PWD/../build-tblgens/bin/llvm-tblgen" \
		-DCLANG_TABLEGEN="$PWD/../build-tblgens/bin/clang-tblgen" \
		-DCLANG_TABLEGEN_EXE="$PWD/../build-tblgens/bin/clang-tblgen" \
		-DCMAKE_BUILD_TYPE=MinSizeRel \
		-DCMAKE_CXX_FLAGS="-I$PWD/../src/llvm/projects/clang/include/ -I$PWD/projects/clang/include/" \
		-DCMAKE_INSTALL_PREFIX="$HOME/cctools/" \
		../src/llvm
	make -j$(nproc --all) install-libtapi install-tapi-headers install-tapi

### Notes:

- We first build llvm- and clang-tblgen as they are required for the build.
- We then build (lib)tapi ourselves instead of relying on the provided build script(s).
  - By doing this ourselves, we can speed up the build significantly by disabling tests and other undesirable components (e.g., architecture backends).
  - Since we built the *-tblgen binaries ourselves, we have to explicitly specify their paths.

### (lib)tapi resources:

* https://github.com/tpoechtrager/apple-libtapi#readme
* https://opensource.apple.com/source/tapi/tapi-1100.0.11/

---

## 6. Build cctools-port with TAPI support

*"[cctools is] a set of essential tools to support development on Mac OS X and Darwin. Conceptually similar to binutils on other platforms."* - [MacPorts](https://ports.macports.org/port/cctools/)

*cctools-port is a port of Apple's cctools and ld64 for Linux and \*BSD* - [Thomas Pöchtrager](https://github.com/tpoechtrager/cctools-port)

### The commands:

	git clone --depth=1 https://github.com/tpoechtrager/cctools-port
	cd cctools-port/cctools
	./configure --prefix="$HOME/cctools/" \
		--target=aarch64-apple-darwin14 \
		--enable-tapi-support \
		--with-libtapi="$HOME/cctools/" \
		--program-prefix="" \
		CC="$HOME/linux/iphone/bin/clang" \
		CXX="$HOME/linux/iphone/bin/clang++" \
		CXXABI_LIB="-l:libc++abi.a" \
		LDFLAGS="-Wl,-rpath,'\$\$ORIGIN/../lib' -Wl,-z,origin"
	make -j$(nproc --all) install
	cp -a $HOME/cctools/* $HOME/my-toolchain/

### Notes:

- We specify the target as aarch64 (arm64) as we will use it to build for iOS.
- We specify the path to (lib)tapi in order to enable support for it.
- We then point to our newly built clang as Apple's clang is required in order to add arm64e support to LD64.
- Additionally, we statically link libc++abi as its version can vary significantly between systems.
- We then tell the linker to adjust the relative library path so the cctools can find (lib)tapi and its resources.

### cctools-port resources:

* https://github.com/tpoechtrager/cctools-port#readme
* https://opensource.apple.com/source/cctools/

---

## 7. Final touches

Uncomment your previously commented out `$LD_LIBRARY_PATH` and/or `$PATH` declarations from your shell's profile and restart your shell.

Remove some or all of the intermediate directories created during the compilation processes (e.g., $HOME/cctools) that are no longer necessary to keep around. The various tool source directories can also be removed if so desired.

Lastly, move the contents of `$HOME/my-toolchain/` to the desired location.

---

And, voilà, you have an iOS toolchain!

~ Lightmann
