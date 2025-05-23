Installation Instructions
==================

These are instructions for building stellar-core from source.

For a potentially quicker set up, the following projects could be good alternatives:

* stellar-core in a [docker container](https://github.com/stellar/docker-stellar-core)
* stellar-core and [horizon](https://github.com/stellar/go/tree/master/services/horizon) in a [docker container](https://github.com/stellar/docker-stellar-core-horizon)
* pre-compiled [packages](https://github.com/stellar/packages)

## Which version to run?

In general, you should aim to run the most recent stable version of core, so make sure
to keep track of new releases.

We _highly_ recommend upgrading to the latest core release _within 30 days of a release_ as
highlighted in our [protocol and security release notes](docs/software/security-protocol-release-notes.md) in case
the release contains security fixes that could be exploited (we do not disclose
ahead of time if a release contains security fixes to give people time to upgrade).

As a consequence, old, potentially insecure or abandoned nodes _running releases that are older than 90 days will get blocked_ by newer nodes (if there are newer releases
of course).

## Picking a version to run

Best is to use the latest *stable* release that can be downloaded from https://github.com/stellar/stellar-core/releases


Alternatively, branches are organized in the following way:

| branch name | description | quality bar |
| ----------- | ----------- | ----------- |
| master      | development branch | all unit tests passing |
| testnet     | version deployed to testnet | acceptance tests passing |
| prod        | version currently deployed on the live network | no recall class issue found in testnet and staging |

For convenience, we also keep a record in the form of release tags of the
 versions that make it to production:
 * pre-releases are versions that get deployed to testnet
 * releases are versions that made it all the way to production

## Containerized dev environment

We maintain a pre-configured Docker configuration ready for development with VSCode.

See the [dev container's README](.devcontainer/README.md) for more detail.

## Runtime dependencies

`stellar-core` does not have many dependencies.

If core was configured (see below) to work with Postgresql, a local Postgresql server
 will need to be deployed to the same host.

To install Postgresql, follow instructions from the [Postgresql download page](https://www.postgresql.org/download/).

## Build Dependencies

- c++ toolchain and headers that supports c++17
    - `clang` >= 12.0
    - `g++` >= 10.0
- `cmake` (version 3.16 or higher) - Required for the CMake build system.
- A C++ build tool supported by CMake, such as `ninja` (recommended for speed) or `make`.
- `pkg-config`
- `bison` and `flex`
- `libpq-dev` unless you disable PostgreSQL support in the build step below.
- 64-bit system
- `clang-format-12` (for `make format` or `ninja format-cpp` to work)
- `sed` and `perl`
- `libunwind-dev`
- Rust toolchain (see [Installing Rust](#installing-rust) subsection)
  - `cargo` >= 1.74
  - `rust` >= 1.74
- **Note for Autotools (Legacy) Builds:** `autoconf`, `automake`, and `libtool` are also required if building with the legacy autotools system.

### Installing Rust

Building the Rust components requires the `cargo` package manager and build system, as well as the `rustc` compiler, both version 1.74 or later.

We recommend installing Rust using the Rust project's `rustup` installer, which can be found on [rustup.rs](https://rustup.rs).

We also include a script in the repository `install-rust.sh` that downloads and runs a known version of `rustup` on x64-linux hosts, such as those used for CI and packaging.

### Ubuntu

#### Ubuntu 20.04
You can install the [test toolchain](#adding-the-test-toolchain) to build and run stellar-core with the latest version of the llvm toolchain.

Alternatively, if you want to just depend on stock Ubuntu, you will have to build with clang *and* have use `libc++` instead of `libstdc++` when compiling.

Ubuntu 20.04 has clang-12 available, that you can install with

    # install clang-12 toolchain
    sudo apt-get install clang-12

After installing packages, head to [building with clang and libc++](#building-with-clang-and-libc).


#### Adding the test toolchain (optional)
    # NOTE: newer version of the compilers are not
    #    provided by stock distributions
    #    and are provided by the /test toolchain
    sudo apt-get install software-properties-common
    sudo add-apt-repository ppa:ubuntu-toolchain-r/test
    sudo apt-get update

#### Installing packages
    # common packages
    sudo apt-get install git build-essential pkg-config cmake ninja-build autoconf automake libtool bison flex libpq-dev libunwind-dev parallel sed perl
    # if using clang
    sudo apt-get install clang-12
    # clang with libstdc++
    sudo apt-get install gcc-10
    # if using g++ or building with libstdc++
    # sudo apt-get install gcc-10 g++-10 cpp-10

In order to make changes, you'll need to install the proper version of clang-format.

In order to install the llvm (clang) toolchain, you may have to follow instructions on https://apt.llvm.org/

    sudo apt-get install clang-format-12

### Ubuntu 24.04 and Newer Linux Versions

Some newer Ubuntu versions have reported issues with older compiler versions. For Ubuntu 24.04 and
other newer Linux distros, it is recommended to build with gcc-13 or clang-18. This can be installed as follows:

```zsh
# if using clang
sudo apt-get install clang-18
# if using g++ or building with libstdc++
# sudo apt-get install gcc-13 g++-13 cpp-13
# if building with libc++
sudo apt-get install libc++-18-dev libc++abi-18-dev
```

Note that installing libc++-18 via apt will uninstall all other libc++ versions.

Additionally, some newer Linux distros no longer package clang-format-12, and newer clang-format versions are
not backwards compatible. To build from source, you'll need to do the following:

```zsh
sudo apt-get install ninja-build cmake
git clone --depth 1 --branch llvmorg-12.0.1 https://github.com/llvm/llvm-project.git
cd llvm-project
sed -i "17i #include <stdint.h>" llvm/include/llvm/Support/Signals.h
CC=clang CXX=clang++ cmake -S llvm -B build -G Ninja -DLLVM_ENABLE_PROJECTS="clang" -DCMAKE_BUILD_TYPE=Release
cd build
ninja clang-format
sudo cp bin/clang-format /usr/bin/clang-format-12
cd ../..
rm -rf llvm-project/
```

### MacOS 
When building on MacOS, here's some dependencies you'll need:
- Install xcode
- Install [Rust](#installing-rust)
- Install [homebrew](https://brew.sh)
- `brew install libsodium libtool autoconf automake pkg-config libpq openssl parallel ccache bison gnu-sed perl coreutils`

You'll also need to configure pkg-config by adding the following to your shell (`.zshenv` or `.zshrc`):
```zsh
export PKG_CONFIG_PATH="$PKG_CONFIG_PATH:$(brew --prefix)/opt/libpq/lib/pkgconfig"
export PKG_CONFIG_PATH="$PKG_CONFIG_PATH:$(brew --prefix)/opt/openssl@3/lib/pkgconfig"
export PATH="$(brew --prefix bison)/bin:$PATH"
```
- Install `clang-format-12` 
> Note: `brew` does not contain a cask for `clang-format-12`, and other versions of `clang-format`  have different formatting behavior.

To install `clang-format-12`, run the commands listed below.
```
# Download clang+llvm 12.0.0 for x86_64 apple darwin.
curl -OL https://github.com/llvm/llvm-project/releases/download/llvmorg-12.0.0/clang+llvm-12.0.0-x86_64-apple-darwin.tar.xz 
# Unarchive the download.
tar -xf clang+llvm-12.0.0-x86_64-apple-darwin.tar.xz 
# Copy the contents of bin/clang-format to /usr/local/bin.
sudo cp -R clang+llvm-12.0.0-x86_64-apple-darwin/bin/clang-format /usr/local/bin
# Ensure "clang-format version 12.0.0" is printed.
clang-format --version
```

> Note: macOS will block the execution for security purposes, open System Preferences > Security & Privacy and allow `clang-format` to run.

### Windows
See [INSTALL-Windows.md](INSTALL-Windows.md)

## Building with Autotools (Legacy)

*Note: CMake is now the recommended build system. See the 'Building with CMake (Recommended)' section below.*

- `git clone https://github.com/stellar/stellar-core.git`
- `cd stellar-core`
- `git submodule init`
- `git submodule update`
- Type `./autogen.sh`.
- Type `./configure`   *(If configure complains about compiler versions, try `CXX=clang-12 ./configure` or `CXX=g++-10 ./configure` or similar, depending on your compiler.)*
- Type `make` or `make -j<N>` (where `<N>` is the number of parallel builds, a number less than the number of CPU cores available, e.g. `make -j3`)
- Type `make check` to run tests.
- Type `make install` to install.

## Building with CMake (Recommended)

CMake is the recommended build system for `stellar-core`.

**1. Prerequisites:**

Ensure you have all the necessary tools and libraries listed in the "Build Dependencies" section, including:
*   A C++17 compliant compiler (Clang >= 12 or GCC >= 10).
*   CMake (version 3.16 or newer).
*   Ninja (recommended) or Make.
*   `pkg-config`
*   `bison` and `flex`
*   `libpq-dev` (unless PostgreSQL support is disabled).
*   `libunwind-dev`
*   Rust toolchain (Cargo >= 1.74, Rust >= 1.74).

**2. Clone Repository and Submodules:**

If you haven't already, clone the repository and initialize submodules:
```bash
git clone https://github.com/stellar/stellar-core.git
cd stellar-core
git submodule update --init --recursive
```

**3. Configure the Build (CMake):**

Create a build directory and run CMake to configure the project. It's recommended to build outside the source directory.
```bash
cmake -S . -B build -G Ninja [OPTIONS]
```
*   `-S .`: Specifies the source directory (current directory).
*   `-B build`: Specifies the build directory (a new directory named `build`).
*   `-G Ninja`: Specifies Ninja as the build system generator. You can use `-G "Unix Makefiles"` for Make.
*   `[OPTIONS]`: CMake options to customize the build (see below).

**Common CMake Options:**

You can pass options to CMake using the `-D<OPTION_NAME>=<VALUE>` syntax. Here are some common options corresponding to the old `./configure` flags:

*   `-DCMAKE_BUILD_TYPE=Release|Debug|RelWithDebInfo`: Standard CMake build types. `Release` for optimized builds, `Debug` for debugging symbols and no optimizations. Defaults typically to `Debug` or none if not set.
*   `-DCMAKE_C_COMPILER=clang|gcc`: Specify C compiler.
*   `-DCMAKE_CXX_COMPILER=clang++|g++`: Specify C++ compiler.
*   `-DDISABLE_POSTGRES=ON|OFF`: Disable/Enable PostgreSQL support (Default: `OFF`).
*   `-DBUILD_TESTS=ON|OFF`: Build the test suite (Default: `ON`). Executable will include test code.
*   `-DENABLE_ASAN=ON|OFF`: Enable AddressSanitizer (Default: `OFF`).
*   `-DENABLE_TSAN=ON|OFF`: Enable ThreadSanitizer (Default: `OFF`).
*   `-DENABLE_MEMCHECK=ON|OFF`: Enable MemorySanitizer (Default: `OFF`).
*   `-DENABLE_UNDEFINEDCHECK=ON|OFF`: Enable UndefinedBehaviorSanitizer (Default: `OFF`).
*   `-DENABLE_EXTRA_CHECKS=ON|OFF`: Enable additional debugging checks (Default: `OFF`).
*   `-DENABLE_TRACY=ON|OFF`: Enable Tracy client integration (Default: `OFF`).
*   `-DENABLE_TRACY_GUI=ON|OFF`: Build the Tracy GUI tool (Default: `OFF`). (Requires additional GUI dependencies like Freetype, GLFW, etc.)
*   `-DENABLE_TRACY_CAPTURE=ON|OFF`: Build the Tracy capture tool (Default: `OFF`).
*   `-DENABLE_TRACY_CSVEXPORT=ON|OFF`: Build the Tracy csvexport tool (Default: `OFF`).
*   `-DENABLE_NEXT_PROTOCOL_VERSION_UNSAFE_FOR_PRODUCTION=ON|OFF`: Enable next protocol version features (UNSAFE FOR PRODUCTION) (Default: `OFF`).
*   `-DUSE_SPDLOG=ON|OFF`: Use bundled spdlog library (Default: `ON`).
*   `-DDISABLE_LIBUNWIND=ON|OFF`: Disable libunwind for backtraces (Default: `OFF`).
*   `-DFORCE_INTERNAL_LIBSODIUM=ON|OFF`: Force build of internal libsodium instead of system version (Default: `OFF`).
*   `-DFORCE_INTERNAL_XDRPP=ON|OFF`: Force build of internal xdrpp instead of system version (Default: `OFF`).

Example: Configure a release build with tests disabled:
```bash
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=OFF
```

**4. Compile the Code:**

After configuration, run the build command:
```bash
cmake --build build --parallel <N>
```
*   `<N>`: Number of parallel build jobs (e.g., number of CPU cores).

The main `stellar-core` executable will be located at `build/src/stellar-core`. Optional Tracy tools (if enabled and built) will also be in the `build/` directory (e.g. `build/tracy-gui`, `build/tracy-capture`, `build/tracy-csvexport`).

**5. Run Tests (if enabled):**

If tests were built (`BUILD_TESTS=ON`, default), you can run them from the build directory:
```bash
cd build
ctest
```
Or, to run a specific test:
```bash
ctest -R stellar-core-selftest-pg
```
To run with memory checking (if Valgrind is available and CTest is configured for it):
```bash
ctest -T memcheck
```

**6. Install (Optional):**

To install the `stellar-core` executable and related files to a specified location:
First, configure with an install prefix:
```bash
cmake -S . -B build -G Ninja -DCMAKE_INSTALL_PREFIX=/usr/local/stellar
```
Then, run the install command:
```bash
cmake --install build
```
The executable will be installed to `/usr/local/stellar/bin/stellar-core`. Documentation and example config files will also be installed under `/usr/local/stellar/share/stellar-core/`.

## Building with clang and libc++

On some systems, building with `libc++`, [LLVM's version of the standard library](https://libcxx.llvm.org/) can be done instead of `libstdc++` (typically used on Linux).

NB: there are newer versions available of both clang and libc++, you will have to use the versions suited for your system.

You may need to install additional packages for this, for example, on Linux Ubuntu 20.04 LTS with clang-12:

    # install libc++ headers
    sudo apt-get install libc++-12-dev libc++abi-12-dev

Here are sample steps to achieve this:

    export CC=clang-12
    export CXX=clang++-12
    export CFLAGS="-O3 -g1 -fno-omit-frame-pointer"
    export CXXFLAGS="$CFLAGS -stdlib=libc++"
    git clone https://github.com/stellar/stellar-core.git
    cd stellar-core/
    ./autogen.sh && ./configure && make -j6

## Building for ARM Linux (i.e. Raspberry Pi)

`stellar-core` is lightweight and can run on many edge devices such as a Raspberry Pi. However, there is currently a
[linker bug](https://bugs.llvm.org/show_bug.cgi?id=16404) in the default ARM `libgcc` runtime, so `compiler-rt` must be used instead.
Here are sample steps to achieve this:

    export CC=clang-12
    export CXX=clang++-12
    export CFLAGS="-O3 -g1 -fno-omit-frame-pointer --rtlib=compiler-rt"
    export CXXFLAGS="$CFLAGS -stdlib=libc++"
    git clone https://github.com/stellar/stellar-core.git
    cd stellar-core/
    ./autogen.sh && ./configure && make -j4

## Building with Tracing

Configuring with `--enable-tracy` will build and embed the client component of the [Tracy](https://github.com/wolfpld/tracy) high-resolution tracing system in the `stellar-core` binary.

The tracing client will activate automatically when stellar-core is running, and will listen for connections from Tracy servers (a command-line capture utility, or a cross-platform GUI).

You do not need to download the tracy server, and will likely run into versioning issues if you do. Instead, the Tracy server components can also be compiled by configuring with `--enable-tracy-gui` or `--enable-tracy-capture`. Once compiled, the tracy server can be started with `./tracy-gui` or `./tracy`, respectively.

The GUI depends on the `capstone`, `freetype` and `glfw` libraries and their headers, and on linux or BSD the `GTK-2.0` libraries and headers. On Windows and MacOS, native toolkits are used instead.


    # On Ubuntu
    $ sudo apt-get install libcapstone-dev libfreetype6-dev libglfw3-dev libgtk2.0-dev

    # On MacOS
    $ brew install capstone freetype2 glfw
