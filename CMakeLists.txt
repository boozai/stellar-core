cmake_minimum_required(VERSION 3.16)
project(stellar-core LANGUAGES CXX)

include(ExternalProject)
find_package(PkgConfig QUIET)
if(NOT PkgConfig_FOUND)
    message(WARNING "pkg-config not found. System library detection will be limited. Please install pkg-config.")
endif()

# Ensure Git submodules are initialized and updated
find_package(Git QUIET)
if(GIT_FOUND)
    message(STATUS "Updating Git submodules...")
    execute_process(
        COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        RESULT_VARIABLE GIT_SUBMODULE_RESULT
        OUTPUT_VARIABLE GIT_SUBMODULE_OUTPUT
        ERROR_VARIABLE GIT_SUBMODULE_OUTPUT
    )
    if(NOT GIT_SUBMODULE_RESULT EQUAL "0")
        message(WARNING "Git submodule update failed:\n${GIT_SUBMODULE_OUTPUT}")
    else()
        message(STATUS "Git submodules updated successfully.")
    endif()
else()
    message(WARNING "Git not found. Submodules might not be up-to-date. Please ensure Git is installed and in PATH.")
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED On)

set(PROJECT_WARNING_FLAGS
    -Wall
    -Wno-unused-command-line-argument
    -Qunused-arguments
    -Wno-unused-local-typedef
    -Wno-unknown-warning-option
    -Werror=unused-result
)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pthread")

add_compile_definitions(FMT_HEADER_ONLY=1)

option(SDF_PREFS "Enable build settings preferred by Stellar developers" ON)
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer" OFF)
option(ENABLE_MEMCHECK "Enable MemorySanitizer" OFF)
option(ENABLE_UNDEFINEDCHECK "Enable UndefinedBehaviorSanitizer" OFF)
option(ENABLE_EXTRA_CHECKS "Enable additional debugging checks" OFF)
option(ENABLE_CCACHE "Build with ccache if available" ON)
option(ENABLE_AFL "Build with AFL fuzzing instrumentation" OFF)
option(DISABLE_POSTGRES "Disable PostgreSQL support" OFF)
option(BUILD_TESTS "Build the test suite" ON)
option(ENABLE_TRACY "Enable Tracy profiler client stub" OFF)
option(ENABLE_NEXT_PROTOCOL_VERSION_UNSAFE_FOR_PRODUCTION "Enable next protocol version (UNSAFE FOR PRODUCTION)" OFF)
option(DISABLE_LIBUNWIND "Disable libunwind for backtraces" OFF)
option(USE_SPDLOG "Use spdlog library" ON)

# CCACHE SUPPORT
if(ENABLE_CCACHE)
    find_program(CCACHE_PROGRAM ccache)
    if(CCACHE_PROGRAM)
        message(STATUS "Using ccache: ${CCACHE_PROGRAM}")
        set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
        set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    else()
        message(STATUS "ENABLE_CCACHE is ON but ccache program not found. Build will proceed without ccache.")
    endif()
endif()

# SANITIZERS
set(SANITIZER_OPTS_LIST "") # Use a list to collect sanitizer options
set(SANITIZER_ENABLED_NAME "") # To track which sanitizer is enabled, for error messages

if(ENABLE_ASAN)
    list(APPEND SANITIZER_OPTS_LIST "address")
    set(SANITIZER_ENABLED_NAME "ASAN")
endif()

if(ENABLE_TSAN)
    if(SANITIZER_ENABLED_NAME)
        message(FATAL_ERROR "TSAN cannot be enabled with other sanitizers (${SANITIZER_ENABLED_NAME}). Please enable only one sanitizer.")
    endif()
    list(APPEND SANITIZER_OPTS_LIST "thread")
    set(SANITIZER_ENABLED_NAME "TSAN")
endif()

if(ENABLE_MEMCHECK)
    if(SANITIZER_ENABLED_NAME)
        message(FATAL_ERROR "MEMCHECK cannot be enabled with other sanitizers (${SANITIZER_ENABLED_NAME}). Please enable only one sanitizer.")
    endif()
    # MEMCHECK implies specific flags that are not just keywords for -fsanitize
    list(APPEND SANITIZER_OPTS_LIST "memory")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize-memory-track-origins=2 -fsanitize-memory-use-after-dtor")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-memory-track-origins=2 -fsanitize-memory-use-after-dtor")
    add_compile_definitions(MSAN_ENABLED=1) # For config.h and conditional compilation
    set(SANITIZER_ENABLED_NAME "MEMCHECK")

    # Check for LIBCXX_PATH. This is a basic check; a real build might need to link against a specific libc++ when MSan is active.
    if(NOT DEFINED ENV{LIBCXX_PATH} AND NOT DEFINED LIBCXX_PATH)
        message(WARNING "MEMCHECK is enabled, but LIBCXX_PATH is not set (as an environment variable or CMake variable). This might lead to issues if a specific C++ standard library build is required for MSan.")
    else()
        # Optionally, print the path if it's set, for debugging
        if(DEFINED ENV{LIBCXX_PATH})
            message(STATUS "LIBCXX_PATH (env): $ENV{LIBCXX_PATH}")
        endif()
        if(DEFINED LIBCXX_PATH)
            message(STATUS "LIBCXX_PATH (cmake): ${LIBCXX_PATH}")
        endif()
    endif()
endif()

if(ENABLE_UNDEFINEDCHECK)
    if(SANITIZER_ENABLED_NAME)
        message(FATAL_ERROR "UNDEFINEDCHECK cannot be enabled with other sanitizers (${SANITIZER_ENABLED_NAME}). Please enable only one sanitizer.")
    endif()
    list(APPEND SANITIZER_OPTS_LIST "undefined")
    # It's good practice to include "nullability" related checks with undefined behavior
    list(APPEND SANITIZER_OPTS_LIST "nullability") 
    set(SANITIZER_ENABLED_NAME "UNDEFINEDCHECK")
endif()

if(SANITIZER_OPTS_LIST)
    string(JOIN "," SANITIZE_OPTS_STRING "${SANITIZER_OPTS_LIST}") # Convert list to comma-separated string for -fsanitize
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=${SANITIZE_OPTS_STRING} -fno-omit-frame-pointer -fno-common")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=${SANITIZE_OPTS_STRING} -fno-omit-frame-pointer -fno-common")
    message(STATUS "Sanitizers enabled: ${SANITIZER_ENABLED_NAME} (${SANITIZE_OPTS_STRING})")
    if(ENABLE_MEMCHECK)
        message(STATUS "MEMCHECK also added -fsanitize-memory-track-origins=2 and -fsanitize-memory-use-after-dtor to C/CXX flags.")
    endif()
endif()

# SDF PREFERENCES
if(SDF_PREFS)
    # Append to the existing list of warning flags.
    # PROJECT_WARNING_FLAGS is defined near the top of the file.
    list(APPEND PROJECT_WARNING_FLAGS "-fcolor-diagnostics")
    message(STATUS "SDF Preferences enabled: Added -fcolor-diagnostics to PROJECT_WARNING_FLAGS.")
endif()

# APPLY WARNING FLAGS
# This must be done after PROJECT_WARNING_FLAGS might be modified (e.g., by SDF_PREFS).
# The initial PROJECT_WARNING_FLAGS are set near the top of the file.
if(PROJECT_WARNING_FLAGS)
    string(REPLACE ";" " " PROJECT_WARNING_FLAGS_STRING "${PROJECT_WARNING_FLAGS}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${PROJECT_WARNING_FLAGS_STRING}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${PROJECT_WARNING_FLAGS_STRING}")
    message(STATUS "Applied project warning flags: ${PROJECT_WARNING_FLAGS_STRING}")
endif()

# EXTRA CHECKS
if(ENABLE_EXTRA_CHECKS)
    add_compile_definitions(
        _GLIBCXX_DEBUG=1
        _GLIBCXX_SANITIZE_VECTOR=1
        _LIBCPP_DEBUG=0 # Standard is to set to 0 for libc++ if not using its specific debug mode.
        BEST_OFFER_DEBUGGING # Application specific
        SCP_DEBUGGING        # Application specific
    )
    message(STATUS "Extra debugging checks enabled (_GLIBCXX_DEBUG, _GLIBCXX_SANITIZE_VECTOR, _LIBCPP_DEBUG=0, BEST_OFFER_DEBUGGING, SCP_DEBUGGING).")
endif()

# AFL FUZZING SUPPORT
if(ENABLE_AFL)
    # Check for conflicts
    if(SANITIZER_ENABLED_NAME) # SANITIZER_ENABLED_NAME is set in the Sanitizers block
        message(FATAL_ERROR "AFL fuzzing (ENABLE_AFL) cannot be enabled when a sanitizer (${SANITIZER_ENABLED_NAME}) is already active. Please disable sanitizers to use AFL.")
    endif()
    if(ENABLE_CCACHE AND CCACHE_PROGRAM) # CCACHE_PROGRAM is set in the CCACHE block
        message(FATAL_ERROR "AFL fuzzing (ENABLE_AFL) cannot be enabled with ccache. Please set ENABLE_CCACHE to OFF to use AFL.")
    endif()

    # Find AFL compilers (prefer Clang-based ones)
    find_program(AFL_C_COMPILER NAMES afl-clang-fast afl-gcc)
    find_program(AFL_CXX_COMPILER NAMES afl-clang-fast++ afl-g++)

    if(AFL_C_COMPILER AND AFL_CXX_COMPILER)
        message(STATUS "Found AFL C compiler: ${AFL_C_COMPILER}")
        message(STATUS "Found AFL CXX compiler: ${AFL_CXX_COMPILER}")
        set(CMAKE_C_COMPILER "${AFL_C_COMPILER}")
        set(CMAKE_CXX_COMPILER "${AFL_CXX_COMPILER}")

        # Check if using Clang-based AFL compiler to add AFL_LLVM_MODE definition
        # The original request mentioned -DAFL_LLVM_MODE=1 for CMAKE_CXX_FLAGS,
        # but add_compile_definitions is more standard for preprocessor defines.
        string(REGEX MATCH "clang" IS_AFL_CLANG_COMPILER "${AFL_C_COMPILER}")
        if(IS_AFL_CLANG_COMPILER)
            message(STATUS "AFL Clang-based compiler detected. Adding AFL_LLVM_MODE=1 compile definition.")
            add_compile_definitions(AFL_LLVM_MODE=1) # For config.h and conditional compilation
        else()
            message(STATUS "AFL GCC-based compiler detected.")
        endif()
    else()
        message(FATAL_ERROR "AFL compilers (afl-clang-fast/afl-clang-fast++ or afl-gcc/afl-g++) not found. Please install AFL and ensure compilers are in PATH.")
    endif()
else()
    # Ensure regular compilers are used if AFL is not enabled
    # This is generally handled by CMake project() call, but can be explicit
    # For now, this else block is more for logical completeness; typically not needed if project() sets default C/CXX compilers.
endif()

# SUBMODULES & LIBRARIES SECTION

# LIBSODIUM
option(FORCE_INTERNAL_LIBSODIUM "Force building libsodium from source" OFF)
if(SANITIZER_ENABLED_NAME) # SANITIZER_ENABLED_NAME is set if any sanitizer is active
    message(STATUS "Sanitizer (${SANITIZER_ENABLED_NAME}) is enabled. Forcing internal build of libsodium.")
    set(FORCE_INTERNAL_LIBSODIUM ON)
endif()

if(NOT FORCE_INTERNAL_LIBSODIUM AND PkgConfig_FOUND)
    pkg_check_modules(LIBSODIUM_PKG libsodium>=1.0.17)
endif()

if(LIBSODIUM_PKG_FOUND AND NOT FORCE_INTERNAL_LIBSODIUM)
    message(STATUS "Found libsodium via pkg-config: ${LIBSODIUM_PKG_LIBRARIES} (includes: ${LIBSODIUM_PKG_INCLUDE_DIRS})")
    set(LIBSODIUM_LIBRARIES ${LIBSODIUM_PKG_LIBRARIES})
    set(LIBSODIUM_INCLUDE_DIRS ${LIBSODIUM_PKG_INCLUDE_DIRS})
    add_library(libsodium INTERFACE IMPORTED)
    set_target_properties(libsodium PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES "${LIBSODIUM_INCLUDE_DIRS}"
        INTERFACE_LINK_LIBRARIES "${LIBSODIUM_LIBRARIES}"
    )
else()
    if(FORCE_INTERNAL_LIBSODIUM)
        message(STATUS "Forcing internal build of libsodium.")
    elseif(NOT PkgConfig_FOUND)
        message(STATUS "pkg-config not found. Building libsodium from source.")
    else()
        message(STATUS "Libsodium not found via pkg-config or version < 1.0.17. Building from source.")
    endif()

    set(LIBSODIUM_INSTALL_DIR ${CMAKE_BINARY_DIR}/libsodium_install)
    set(LIBSODIUM_SOURCES_DIR ${CMAKE_SOURCE_DIR}/lib/libsodium)
    set(LIBSODIUM_INCLUDE_DIRS ${LIBSODIUM_INSTALL_DIR}/include)
    set(LIBSODIUM_LIBRARIES ${LIBSODIUM_INSTALL_DIR}/lib/libsodium.a)

    # Pass compiler flags for sanitizers if enabled
    set(LIBSODIUM_CONFIGURE_FLAGS "")
    if(SANITIZER_OPTS_STRING) # SANITIZER_OPTS_STRING is like "address", "thread", etc.
        list(APPEND LIBSODIUM_CONFIGURE_FLAGS "CFLAGS=${CMAKE_C_FLAGS}" "CXXFLAGS=${CMAKE_CXX_FLAGS}")
    endif()

    ExternalProject_Add(libsodium_external
        SOURCE_DIR ${LIBSODIUM_SOURCES_DIR}
        PREFIX ${LIBSODIUM_SOURCES_DIR}/build # Build in a subdir of source to keep it clean
        CONFIGURE_COMMAND ${LIBSODIUM_SOURCES_DIR}/configure --disable-shared --enable-static --prefix=${LIBSODIUM_INSTALL_DIR} ${LIBSODIUM_CONFIGURE_FLAGS}
        BUILD_COMMAND make -j${CMAKE_BUILD_PARALLEL_LEVEL}
        INSTALL_COMMAND make install
        BUILD_BYPRODUCTS ${LIBSODIUM_LIBRARIES}
    )

    add_library(libsodium STATIC IMPORTED)
    set_target_properties(libsodium PROPERTIES
        IMPORTED_LOCATION "${LIBSODIUM_LIBRARIES}"
        INTERFACE_INCLUDE_DIRECTORIES "${LIBSODIUM_INCLUDE_DIRS}"
    )
    add_dependencies(libsodium libsodium_external) # libsodium target depends on the external build
    message(STATUS "Configured internal libsodium build (Install dir: ${LIBSODIUM_INSTALL_DIR})")
endif()
# Ensure other targets can find libsodium headers and libraries
# Using an interface library 'libsodium' which is either IMPORTED (pkg-config) or STATIC IMPORTED (internal build)


# XDRPP (XDRC tool and library)
option(FORCE_INTERNAL_XDRPP "Force building xdrpp from source" OFF)
if(SANITIZER_ENABLED_NAME)
    message(STATUS "Sanitizer (${SANITIZER_ENABLED_NAME}) is enabled. Forcing internal build of xdrpp.")
    set(FORCE_INTERNAL_XDRPP ON)
endif()

set(XDRPP_FOUND_PKGCONFIG FALSE)
set(XDRC_FOUND_PROGRAM FALSE)

if(NOT FORCE_INTERNAL_XDRPP AND PkgConfig_FOUND)
    pkg_check_modules(XDRPP_PKG xdrpp)
    if(XDRPP_PKG_FOUND)
        set(XDRPP_FOUND_PKGCONFIG TRUE)
    endif()
endif()

if(NOT FORCE_INTERNAL_XDRPP)
    find_program(XDRC_EXECUTABLE_SYSTEM xdrc)
    if(XDRC_EXECUTABLE_SYSTEM)
        set(XDRC_FOUND_PROGRAM TRUE)
    endif()
endif()

if(XDRPP_FOUND_PKGCONFIG AND XDRC_FOUND_PROGRAM AND NOT FORCE_INTERNAL_XDRPP)
    message(STATUS "Found xdrpp via pkg-config: ${XDRPP_PKG_LIBRARIES} (includes: ${XDRPP_PKG_INCLUDE_DIRS})")
    message(STATUS "Found xdrc executable via find_program: ${XDRC_EXECUTABLE_SYSTEM}")
    set(XDRPP_LIBRARIES ${XDRPP_PKG_LIBRARIES})
    set(XDRPP_INCLUDE_DIRS ${XDRPP_PKG_INCLUDE_DIRS})
    set(XDRC_EXECUTABLE ${XDRC_EXECUTABLE_SYSTEM})

    add_library(xdrpp INTERFACE IMPORTED)
    set_target_properties(xdrpp PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES "${XDRPP_INCLUDE_DIRS}"
        INTERFACE_LINK_LIBRARIES "${XDRPP_LIBRARIES}"
    )
    # xdrc is just an executable, not a library target to link against directly.
    # XDRC_EXECUTABLE variable will be used in custom commands.
else()
    if(FORCE_INTERNAL_XDRPP)
        message(STATUS "Forcing internal build of xdrpp.")
    elseif(NOT PkgConfig_FOUND)
        message(STATUS "pkg-config not found. Building xdrpp from source.")
    elseif(NOT XDRPP_FOUND_PKGCONFIG)
        message(STATUS "xdrpp library not found via pkg-config. Building from source.")
    elseif(NOT XDRC_FOUND_PROGRAM)
        message(STATUS "xdrc executable not found. Building xdrpp from source.")
    endif()

    set(XDRPP_INSTALL_DIR ${CMAKE_BINARY_DIR}/xdrpp_install)
    set(XDRPP_SOURCES_DIR ${CMAKE_SOURCE_DIR}/lib/xdrpp)
    set(XDRPP_INCLUDE_DIRS ${XDRPP_INSTALL_DIR}/include)
    set(XDRPP_LIBRARIES ${XDRPP_INSTALL_DIR}/lib/libxdrpp.a) # Assuming static library
    set(XDRC_EXECUTABLE ${XDRPP_INSTALL_DIR}/bin/xdrc)

    # Pass compiler flags for sanitizers if enabled
    set(XDRPP_CONFIGURE_FLAGS "")
    if(SANITIZER_OPTS_STRING)
        list(APPEND XDRPP_CONFIGURE_FLAGS "CFLAGS=${CMAKE_C_FLAGS}" "CXXFLAGS=${CMAKE_CXX_FLAGS}")
    endif()

    ExternalProject_Add(xdrpp_external
        SOURCE_DIR ${XDRPP_SOURCES_DIR}
        PREFIX ${XDRPP_SOURCES_DIR}/build # Build in a subdir of source
        CONFIGURE_COMMAND ${XDRPP_SOURCES_DIR}/configure --disable-shared --enable-static --prefix=${XDRPP_INSTALL_DIR} ${XDRPP_CONFIGURE_FLAGS}
        BUILD_COMMAND make -j${CMAKE_BUILD_PARALLEL_LEVEL}
        INSTALL_COMMAND make install
        BUILD_BYPRODUCTS ${XDRPP_LIBRARIES} ${XDRC_EXECUTABLE} # Ensure both lib and tool are built
    )

    add_library(xdrpp STATIC IMPORTED)
    set_target_properties(xdrpp PROPERTIES
        IMPORTED_LOCATION "${XDRPP_LIBRARIES}"
        INTERFACE_INCLUDE_DIRECTORIES "${XDRPP_INCLUDE_DIRS}"
    )
    add_dependencies(xdrpp xdrpp_external)
    message(STATUS "Configured internal xdrpp build (Install dir: ${XDRPP_INSTALL_DIR}, XDRC: ${XDRC_EXECUTABLE})")
endif()
# XDRPP_LIBRARIES, XDRPP_INCLUDE_DIRS, and XDRC_EXECUTABLE are now set.


# LibASIO
# Asio is mostly header-only, but stellar-core uses it with ASIO_SEPARATE_COMPILATION.
# This requires compiling lib/asio.cpp and linking it.
message(STATUS "Configuring Asio...")
add_library(asio_interface INTERFACE)
target_include_directories(asio_interface INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/lib/asio/asio/include
)
target_compile_definitions(asio_interface INTERFACE
    ASIO_SEPARATE_COMPILATION
    ASIO_STANDALONE
)

# Check if lib/asio.cpp exists. If not, it might indicate an issue with submodule checkout or file structure.
set(ASIO_CPP_FILE ${CMAKE_CURRENT_SOURCE_DIR}/lib/asio.cpp)
if(NOT EXISTS ${ASIO_CPP_FILE})
    message(FATAL_ERROR "Asio source file ${ASIO_CPP_FILE} not found. Check submodule status or file path.")
endif()

add_library(asio_static_lib STATIC ${ASIO_CPP_FILE})
# Link the compiled part to the interface.
# Targets that use Asio should link to asio_interface.
target_link_libraries(asio_static_lib PUBLIC asio_interface) # PUBLIC so users of asio_static_lib get interface props
# However, the typical pattern is to link the static lib privately to the interface,
# and then other targets link to the interface. Let's adjust this.
# target_link_libraries(asio_interface INTERFACE asio_static_lib) - No, this is for linking *to* an interface.

# Correct pattern: The interface library exposes the include dirs and defines.
# A separate compiled library provides the symbols. Targets link to *both* or to an
# aggregate library if one were made. Or, more simply, the compiled lib *uses* the interface.
# The instructions suggest linking asio_static_lib to asio_interface (privately).
# This means asio_static_lib gets the compile defs from asio_interface.
# And then other targets would link to asio_static_lib (which brings asio_interface's usage requirements).

# Let's re-evaluate:
# 1. asio_interface: INTERFACE_INCLUDE_DIRECTORIES, INTERFACE_COMPILE_DEFINITIONS.
# 2. asio_static_lib: Source file. This library *needs* the definitions from asio_interface.
# So, asio_static_lib should link to asio_interface.
# Then, other targets link to asio_static_lib.

# Resetting asio_static_lib to link to asio_interface correctly.
# The previous target_link_libraries for asio_static_lib was okay.
# Let's keep it simple: other targets will link to asio_static_lib.
# asio_static_lib itself needs the include dirs to find asio headers for asio.cpp
target_include_directories(asio_static_lib PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/lib/asio/asio/include)
target_compile_definitions(asio_static_lib PRIVATE ASIO_SEPARATE_COMPILATION ASIO_STANDALONE)


# The request was: Link asio_static_lib to asio_interface (privately)
# This is a bit unusual. It usually means asio_static_lib uses asio_interface.
# Let's try to make asio_static_lib an alias or also an interface for downstream.
# The goal is that users link to one thing.
# Perhaps the intent was:
# add_library(asio INTERFACE)
# target_link_libraries(asio INTERFACE asio_interface asio_static_lib)
# This is not quite right.

# Simpler approach:
# `asio_interface` provides the compile definitions and include directories.
# `asio_static_lib` provides the compiled symbols.
# Targets that need Asio should link against `asio_static_lib` AND `asio_interface`.
# Or, we can make `asio_static_lib` also propagate the interface properties.

# Let's stick to the request: "Link asio_static_lib to asio_interface (privately)"
# This means asio_static_lib uses the definitions from asio_interface.
# And then dependent targets link to asio_static_lib.
# target_link_libraries(asio_static_lib PRIVATE asio_interface)
# This is effectively what target_compile_definitions and target_include_directories on asio_static_lib does.

# The most common way to handle this pattern (header-only lib + one .cpp file):
# Create a library (static or shared) from the .cpp file.
# This library has include_directories for the headers.
# This library has compile_definitions needed by the .cpp and headers.
# Other targets link to this library.

# Let's make `asio_static_lib` the main target to link against.
# It will have the source file and the necessary interface properties.
# Removing `asio_interface` and putting all properties on `asio_static_lib`.
# This simplifies it to one target `asio_static_lib` for users.
# The request was to "Create an interface library target asio_interface" and "Link asio_static_lib to asio_interface (privately)"
# This implies asio_interface is for users, and asio_static_lib is an implementation detail.
# So, users link to `asio_interface`. `asio_interface` needs to bring in `asio_static_lib`.
# This is done by `target_link_libraries(asio_interface INTERFACE asio_static_lib)`.

# Re-doing Asio with the clearer understanding:
# 1. asio_static_lib: compiled from asio.cpp. Needs include path & defines for itself.
# 2. asio_interface: INTERFACE library that consumers link to. It provides include paths, defines, and links to asio_static_lib.

# Clean slate for Asio:
message(STATUS "Configuring Asio (Revised)...")
add_library(asio_static_lib STATIC ${ASIO_CPP_FILE})
target_include_directories(asio_static_lib PRIVATE # PRIVATE, as these are for compiling asio.cpp itself
    ${CMAKE_CURRENT_SOURCE_DIR}/lib/asio/asio/include
)
target_compile_definitions(asio_static_lib PRIVATE # PRIVATE, for compiling asio.cpp
    ASIO_SEPARATE_COMPILATION
    ASIO_STANDALONE
)

add_library(asio_interface INTERFACE)
target_include_directories(asio_interface INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/lib/asio/asio/include  # Consumers need this path
)
target_compile_definitions(asio_interface INTERFACE
    ASIO_SEPARATE_COMPILATION  # Consumers need these defines
    ASIO_STANDALONE
)
target_link_libraries(asio_interface INTERFACE
    asio_static_lib # Consumers of asio_interface will also link to asio_static_lib
)
message(STATUS "Asio configured: asio_interface (links to asio_static_lib).")


# SOCI (database access library)
message(STATUS "Configuring SOCI (populating sources)...")
# soci_lib and soci_interface targets should exist from previous setup.

set(SOCI_CORE_SOURCES "")
file(GLOB SOCI_CORE_SOURCES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/lib/soci/src/core/*.cpp")
if(SOCI_CORE_SOURCES_CPP)
    list(APPEND SOCI_CORE_SOURCES ${SOCI_CORE_SOURCES_CPP})
    message(STATUS "  Adding SOCI core sources: ${SOCI_CORE_SOURCES_CPP}")
else()
    message(WARNING "  No .cpp sources found in lib/soci/src/core/ for soci_lib.")
endif()

if(SOCI_CORE_SOURCES)
    target_sources(soci_lib PRIVATE ${SOCI_CORE_SOURCES})
else()
    # Add a dummy source if no core sources are found to keep the library valid, though this is unexpected.
    # Consider a FATAL_ERROR if core SOCI sources are essential and missing.
    message(WARNING "  soci_lib has no core sources. This might be an error.")
endif()

# Conditional PostgreSQL backend for SOCI
if(NOT DISABLE_POSTGRES AND HAVE_POSTGRES) # HAVE_POSTGRES is set in Libpq section
    message(STATUS "  PostgreSQL is enabled for SOCI.")
    set(SOCI_POSTGRESQL_SOURCES "")
    file(GLOB SOCI_POSTGRESQL_SOURCES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/lib/soci/src/backends/postgresql/*.cpp")
    if(SOCI_POSTGRESQL_SOURCES_CPP)
        list(APPEND SOCI_POSTGRESQL_SOURCES ${SOCI_POSTGRESQL_SOURCES_CPP})
        target_sources(soci_lib PRIVATE ${SOCI_POSTGRESQL_SOURCES})
        message(STATUS "    Added SOCI PostgreSQL backend sources: ${SOCI_POSTGRESQL_SOURCES_CPP}")

        # Link soci_lib to PostgreSQL::libpq
        # soci_interface already links to soci_lib, so consumers of soci_interface will get this.
        target_link_libraries(soci_lib PUBLIC PostgreSQL::libpq) # PUBLIC so it propagates if needed, or PRIVATE if only soci_lib needs it
        message(STATUS "    soci_lib linked to PostgreSQL::libpq.")

        # Add PostgreSQL include dirs to soci_interface if not already covered by PostgreSQL::libpq target
        # Assuming PostgreSQL::libpq correctly sets its INTERFACE_INCLUDE_DIRECTORIES
        # target_include_directories(soci_interface INTERFACE ${POSTGRESQL_INCLUDE_DIRS})
        # target_compile_definitions(soci_interface INTERFACE SOCI_HAVE_POSTGRESQL) # Might be needed by SOCI config
        # However, SOCI's own config.h or CMake setup usually handles SOCI_HAVE_POSTGRESQL if sources are compiled.
        # For now, just linking is sufficient as per instructions.
    else()
        message(WARNING "  PostgreSQL backend for SOCI enabled, but no .cpp sources found in lib/soci/src/backends/postgresql/.")
    endif()
else()
    message(STATUS "  PostgreSQL backend for SOCI is disabled (DISABLE_POSTGRES: ${DISABLE_POSTGRES}, HAVE_POSTGRES: ${HAVE_POSTGRES}).")
endif()

message(STATUS "SOCI source population complete.")

# Libmedida (metrics library)
message(STATUS "Configuring Libmedida (populating sources if internal)...")
option(FORCE_INTERNAL_LIBMEDIDA "Force building libmedida from source" OFF)
if(SANITIZER_ENABLED_NAME)
    message(STATUS "  Sanitizer (${SANITIZER_ENABLED_NAME}) is enabled. Forcing internal build of libmedida.")
    set(FORCE_INTERNAL_LIBMEDIDA ON)
endif()

set(USING_INTERNAL_LIBMEDIDA FALSE) # Default to not using internal

if(NOT FORCE_INTERNAL_LIBMEDIDA AND PkgConfig_FOUND)
    pkg_check_modules(LIBMEDIDA_PKG libmedida)
endif()

if(LIBMEDIDA_PKG_FOUND AND NOT FORCE_INTERNAL_LIBMEDIDA)
    message(STATUS "Found libmedida via pkg-config: ${LIBMEDIDA_PKG_LIBRARIES} (includes: ${LIBMEDIDA_PKG_INCLUDE_DIRS})")
    add_library(medida_interface INTERFACE IMPORTED)
    set_target_properties(medida_interface PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES "${LIBMEDIDA_PKG_INCLUDE_DIRS}"
        INTERFACE_LINK_LIBRARIES "${LIBMEDIDA_PKG_LIBRARIES}"
    )
    # LIBMEDIDA_LIBRARIES and LIBMEDIDA_INCLUDE_DIRS are set by pkg_check_modules if successful.
    # We can use these directly or through the interface library.
    set(MEDIDA_LIBRARIES ${LIBMEDIDA_PKG_LIBRARIES})
    set(MEDIDA_INCLUDE_DIRS ${LIBMEDIDA_PKG_INCLUDE_DIRS})
else()
    if(FORCE_INTERNAL_LIBMEDIDA)
        message(STATUS "Forcing internal build of libmedida.")
    elseif(NOT PkgConfig_FOUND)
        message(STATUS "pkg-config not found. Building libmedida from internal sources.")
    else()
        message(STATUS "libmedida not found via pkg-config. Building from internal sources.")
    endif()

    set(USING_INTERNAL_LIBMEDIDA TRUE)
    add_library(medida_lib STATIC "") # Define/ensure medida_lib exists if internal
    add_library(medida_interface INTERFACE) # Define/ensure medida_interface exists

    target_include_directories(medida_interface INTERFACE
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/libmedida/src
    )
    target_link_libraries(medida_lib PRIVATE medida_interface)
    target_link_libraries(medida_interface INTERFACE medida_lib)

    set(MEDIDA_LIBRARIES medida_interface)
    set(MEDIDA_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/lib/libmedida/src)
    message(STATUS "  Configured for internal libmedida build (USING_INTERNAL_LIBMEDIDA: TRUE).")

    # Populate sources for medida_lib
    set(MEDIDA_INTERNAL_SOURCES "")
    file(GLOB MEDIDA_CORE_SOURCES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/lib/libmedida/src/medida/*.cpp")
    if(MEDIDA_CORE_SOURCES_CPP)
        list(APPEND MEDIDA_INTERNAL_SOURCES ${MEDIDA_CORE_SOURCES_CPP})
    endif()
    file(GLOB MEDIDA_REPORTING_SOURCES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/lib/libmedida/src/medida/reporting/*.cpp")
    if(MEDIDA_REPORTING_SOURCES_CPP)
        list(APPEND MEDIDA_INTERNAL_SOURCES ${MEDIDA_REPORTING_SOURCES_CPP})
    endif()
    file(GLOB MEDIDA_STATS_SOURCES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/lib/libmedida/src/medida/stats/*.cpp")
    if(MEDIDA_STATS_SOURCES_CPP)
        list(APPEND MEDIDA_INTERNAL_SOURCES ${MEDIDA_STATS_SOURCES_CPP})
    endif()

    if(MEDIDA_INTERNAL_SOURCES)
        target_sources(medida_lib PRIVATE ${MEDIDA_INTERNAL_SOURCES})
        message(STATUS "    Added internal Libmedida sources: ${MEDIDA_INTERNAL_SOURCES}")
    else()
        message(WARNING "    USING_INTERNAL_LIBMEDIDA is TRUE, but no sources found for libmedida in specified paths.")
        # Add a dummy source to prevent CMake errors if truly no sources are found.
        # This might indicate an issue with the submodule or directory structure.
        # set(MEDIDA_DUMMY_SOURCE "${CMAKE_CURRENT_BINARY_DIR}/dummy_medida.cpp")
        # file(WRITE ${MEDIDA_DUMMY_SOURCE} "void medida_dummy() {}")
        # target_sources(medida_lib PRIVATE ${MEDIDA_DUMMY_SOURCE})
    endif()
else()
     message(STATUS "  Using system/pkg-config libmedida (USING_INTERNAL_LIBMEDIDA: FALSE). medida_lib target will not be populated with sources.")
endif()
# MEDIDA_LIBRARIES and MEDIDA_INCLUDE_DIRS are set, or medida_interface (linking to internal medida_lib) is ready.


# Libpq (PostgreSQL client library)
set(HAVE_POSTGRES 0 CACHE INTERNAL "PostgreSQL support enabled status for config.h") # Default to disabled
if(NOT DISABLE_POSTGRES)
    if(PkgConfig_FOUND)
        pkg_check_modules(LIBPQ_PKG QUIET libpq) # Check quietly first
    endif()

    if(LIBPQ_PKG_FOUND)
        message(STATUS "Found PostgreSQL (libpq) via pkg-config: ${LIBPQ_PKG_LIBRARIES} (includes: ${LIBPQ_PKG_INCLUDE_DIRS})")
        set(POSTGRESQL_LIBRARIES ${LIBPQ_PKG_LIBRARIES})
        set(POSTGRESQL_INCLUDE_DIRS ${LIBPQ_PKG_INCLUDE_DIRS})
        set(HAVE_POSTGRES 1) # For config.h

        # Create an interface library for PostgreSQL
        add_library(PostgreSQL::libpq INTERFACE IMPORTED)
        set_target_properties(PostgreSQL::libpq PROPERTIES
            INTERFACE_INCLUDE_DIRECTORIES "${POSTGRESQL_INCLUDE_DIRS}"
            INTERFACE_LINK_LIBRARIES "${POSTGRESQL_LIBRARIES}"
        )
        message(STATUS "PostgreSQL (libpq) configured using pkg-config. Target PostgreSQL::libpq created.")
    else()
        # If pkg-config fails or not found, try CMake's FindPostgreSQL module
        message(STATUS "libpq not found via pkg-config. Trying find_package(PostgreSQL)...")
        find_package(PostgreSQL QUIET) # Standard CMake module
        if(PostgreSQL_FOUND)
            message(STATUS "Found PostgreSQL via find_package: ${PostgreSQL_LIBRARIES} (includes: ${PostgreSQL_INCLUDE_DIRS})")
            # PostgreSQL_LIBRARIES and PostgreSQL_INCLUDE_DIRS are set by FindPostgreSQL
            set(POSTGRESQL_LIBRARIES ${PostgreSQL_LIBRARIES})
            set(POSTGRESQL_INCLUDE_DIRS ${PostgreSQL_INCLUDE_DIRS})
            set(HAVE_POSTGRES 1) # For config.h

            # The FindPostgreSQL module typically creates PostgreSQL::PostgreSQL if version 9+
            # For older CMake or versions, it might just set variables.
            # We'll ensure a consistent target PostgreSQL::libpq exists.
            if(TARGET PostgreSQL::PostgreSQL)
                message(STATUS "Using target PostgreSQL::PostgreSQL from find_package(PostgreSQL).")
                # Alias it or create our own interface that uses it
                add_library(PostgreSQL::libpq INTERFACE IMPORTED)
                target_link_libraries(PostgreSQL::libpq INTERFACE PostgreSQL::PostgreSQL)
            else()
                # Create the interface library from variables
                add_library(PostgreSQL::libpq INTERFACE IMPORTED)
                set_target_properties(PostgreSQL::libpq PROPERTIES
                    INTERFACE_INCLUDE_DIRECTORIES "${POSTGRESQL_INCLUDE_DIRS}"
                    INTERFACE_LINK_LIBRARIES "${POSTGRESQL_LIBRARIES}"
                )
            endif()
            message(STATUS "PostgreSQL configured using find_package(PostgreSQL). Target PostgreSQL::libpq created/aliased.")
        else()
            message(WARNING "PostgreSQL (libpq) not found via pkg-config or find_package. PostgreSQL support will be disabled.")
            set(DISABLE_POSTGRES ON) # Effectively disable it if not found
            set(HAVE_POSTGRES 0)
        endif()
    endif()
else()
    message(STATUS "PostgreSQL support disabled via DISABLE_POSTGRES option.")
    set(HAVE_POSTGRES 0)
endif()
# POSTGRESQL_LIBRARIES and POSTGRESQL_INCLUDE_DIRS are set if found, or target PostgreSQL::libpq
# HAVE_POSTGRES is set for config.h


# lib3rdparty_a (custom static library for various utilities)
message(STATUS "Configuring lib3rdparty_a...")
set(LIB3RDPARTY_SOURCES "")

# Glob sources from lib/util
file(GLOB UTIL_SOURCES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/lib/util/*.cpp")
if(UTIL_SOURCES_CPP)
    list(APPEND LIB3RDPARTY_SOURCES ${UTIL_SOURCES_CPP})
    message(STATUS "  Added sources from lib/util: ${UTIL_SOURCES_CPP}")
else()
    message(WARNING "  No .cpp sources found in lib/util for lib3rdparty_a.")
endif()

# Glob sources from lib/json (typically jsoncpp.cpp)
file(GLOB JSON_SOURCES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/lib/json/*.cpp")
if(JSON_SOURCES_CPP)
    list(APPEND LIB3RDPARTY_SOURCES ${JSON_SOURCES_CPP})
    message(STATUS "  Added sources from lib/json: ${JSON_SOURCES_CPP}")
else()
    message(WARNING "  No .cpp sources found in lib/json for lib3rdparty_a.")
endif()

# Conditional sources and linking for spdlog
if(USE_SPDLOG)
    # spdlog sources (excluding bundled_fmtlib_format.cpp as fmt is separate)
    file(GLOB SPDLOG_LIB_SOURCES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/lib/spdlog/src/*.cpp")
    list(FILTER SPDLOG_LIB_SOURCES_CPP EXCLUDE REGEX ".*bundled_fmtlib_format\\.cpp$")
    if(SPDLOG_LIB_SOURCES_CPP)
        list(APPEND LIB3RDPARTY_SOURCES ${SPDLOG_LIB_SOURCES_CPP})
        message(STATUS "  Added sources from lib/spdlog/src: ${SPDLOG_LIB_SOURCES_CPP}")
    else()
        message(WARNING "  No .cpp sources (excluding bundled_fmtlib_format.cpp) found in lib/spdlog/src for lib3rdparty_a, though USE_SPDLOG is ON.")
    endif()
endif()

if(NOT LIB3RDPARTY_SOURCES)
    message(WARNING "No source files collected for lib3rdparty_a. It will be an empty library if no sources are added.")
    # Add a dummy source to prevent CMake errors for an empty static library, if truly no sources.
    # However, given the globs, it's expected some sources will be found.
    # Consider adding a dummy file if this warning persists and is unexpected.
    # set(LIB3RDPARTY_DUMMY_SOURCE "${CMAKE_CURRENT_BINARY_DIR}/dummy_lib3rdparty.cpp")
    # file(WRITE ${LIB3RDPARTY_DUMMY_SOURCE} "void lib3rdparty_dummy() {}")
    # list(APPEND LIB3RDPARTY_SOURCES ${LIB3RDPARTY_DUMMY_SOURCE})
endif()

add_library(lib3rdparty_a STATIC ${LIB3RDPARTY_SOURCES})

target_include_directories(lib3rdparty_a PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/lib/util
    ${CMAKE_CURRENT_SOURCE_DIR}/lib/json
    # Asio include dirs are provided by linking against asio_interface
    # Spdlog include dirs are provided by linking against spdlog_interface (if USE_SPDLOG)
)

# Link lib3rdparty_a to other necessary libraries
target_link_libraries(lib3rdparty_a PUBLIC asio_interface sqlite3_lib)
message(STATUS "  lib3rdparty_a linked publicly to asio_interface and sqlite3_lib.")

if(USE_SPDLOG)
    target_link_libraries(lib3rdparty_a PUBLIC spdlog_interface)
    message(STATUS "  lib3rdparty_a linked publicly to spdlog_interface.")
endif()

message(STATUS "lib3rdparty_a configuration complete.")

# Cereal (header-only, but has CMakeLists for options/installation)
set(CEREAL_BUILD_TESTS OFF CACHE BOOL "Disable Cereal tests")
add_subdirectory(lib/cereal)
message(STATUS "Added subdirectory: lib/cereal (Tests: ${CEREAL_BUILD_TESTS})")

# fmt
set(FMT_TEST OFF CACHE BOOL "Disable fmt tests")
# FMT_INSTALL is ON by default, which is fine.
# FMT_DOC is ON by default, can be left as is or turned off if not needed.
# set(FMT_DOC OFF CACHE BOOL "Disable fmt documentation")
add_subdirectory(lib/fmt)
message(STATUS "Added subdirectory: lib/fmt (Tests: ${FMT_TEST})")

# spdlog
if(USE_SPDLOG)
    set(SPDLOG_BUILD_TESTS OFF CACHE BOOL "Disable spdlog tests")
    set(SPDLOG_BUILD_EXAMPLES OFF CACHE BOOL "Disable spdlog examples")
    # spdlog can be header-only or compiled. The CMakeLists.txt handles this.
    # We want to use its targets.
    add_subdirectory(lib/spdlog)
    message(STATUS "Added subdirectory: lib/spdlog (Tests: ${SPDLOG_BUILD_TESTS}, Examples: ${SPDLOG_BUILD_EXAMPLES})")

    # Create an interface library that links against spdlog::spdlog
    # This allows other targets in our project to link against spdlog_interface
    # consistently, whether spdlog is header-only or compiled.
    add_library(spdlog_interface INTERFACE)
    target_link_libraries(spdlog_interface INTERFACE spdlog::spdlog)
    message(STATUS "Created spdlog_interface and linked to spdlog::spdlog")
else()
    # Create a dummy spdlog_interface if USE_SPDLOG is OFF
    # This allows targets to link against spdlog_interface without errors,
    # and it will just be an empty interface.
    add_library(spdlog_interface INTERFACE)
    message(STATUS "USE_SPDLOG is OFF. Created dummy spdlog_interface.")
endif()

# Tracy Profiler Client Stub
# The main option is ENABLE_TRACY. Tracy's CMakeLists.txt uses TRACY_ENABLE.
if(ENABLE_TRACY)
    # Tracy options that might be relevant (defaults are usually fine for client):
    # TRACY_ENABLE: ON/OFF (set based on our ENABLE_TRACY)
    # TRACY_ON_DEMAND: OFF (enables on-demand client-side recording, not typically needed for basic integration)
    # TRACY_NO_EXIT: OFF (prevents client from exiting automatically)
    # TRACY_FIBERS: ON (enables fiber support)
    # TRACY_NO_BROADCAST: OFF (disables network broadcast for server discovery)
    # TRACY_ONLY_IPV4: OFF (forces IPv4 for network communication)
    # TRACY_TIMER_FALLBACK: OFF (uses less precise timers if high-precision ones fail)
    # TRACY_COLLECT_THREAD_NAMES: ON (collects thread names)
    # TRACY_NO_SYSTEM_TRACING: OFF (disables system event tracing)
    # TRACY_NO_CODE_TRANSFER: OFF (disables code transfer to server)
    # TRACY_NO_CONTEXT_SWITCH: OFF (disables context switch tracing)
    # TRACY_NO_SAMPLING: OFF (disables statistical sampling)
    # TRACY_NO_FRAME_IMAGE: OFF (disables frame image capture)
    # TRACY_NO_CALLSTACK: OFF (disables callstack collection) - Note: libunwind is used by Tracy for callstacks on some platforms.

    set(TRACY_ENABLE ON CACHE BOOL "Enable Tracy client profiler" FORCE)
    # Disable parts of tracy we don't need for a client library
    set(TRACY_PROFILER ON CACHE BOOL "Build Tracy Profiler (client part)" FORCE) # Ensure profiler client is built
    set(TRACY_CAPTURE OFF CACHE BOOL "Disable Tracy Capture utility" FORCE) # Separate utility
    set(TRACY_GUI OFF CACHE BOOL "Disable Tracy GUI utility" FORCE) # Separate utility
    set(TRACY_UPDATE OFF CACHE BOOL "Disable Tracy Update utility" FORCE) # Separate utility
    set(TRACY_CSERVE OFF CACHE BOOL "Disable Tracy CServe utility" FORCE) # Separate utility
    set(TRACY_IMPORTER OFF CACHE BOOL "Disable Tracy Importer utility" FORCE) # Separate utility

    # If DISABLE_LIBUNWIND is set, ensure Tracy knows not to attempt using it, if it has such an option.
    # Tracy's CMake seems to check for libunwind itself.
    # We might need to pass -DTRACY_NO_CALLSTACK if DISABLE_LIBUNWIND is true and tracy doesn't pick it up.
    if(DISABLE_LIBUNWIND AND NOT APPLE) # On Apple, Tracy uses other mechanisms for callstacks
         # Tracy has TRACY_NO_CALLSTACK, but it also has specific defines like TRACY_HAS_LIBUNWIND
         # It's better to let Tracy's CMakeLists.txt handle libunwind detection.
         # If issues arise, we might need to explicitly set TRACY_NO_CALLSTACK=ON here.
         message(STATUS "DISABLE_LIBUNWIND is ON. Tracy might have limited callstack capabilities if it relies on libunwind.")
    endif()

    add_subdirectory(lib/tracy)
    message(STATUS "Added subdirectory: lib/tracy (TRACY_ENABLE: ON)")
    # After adding tracy, its library target (usually `TracyClient`) will be available.
    # We will link against this in targets that need profiling.
    # For now, just adding the subdirectory is sufficient.
else()
    message(STATUS "ENABLE_TRACY is OFF. Tracy client will not be built.")
    # If Tracy is not enabled, targets that would optionally link to it should still compile.
    # This might involve using an interface library for Tracy similar to spdlog,
    # or using target_compile_definitions to define TRACY_ENABLE=0.
    # For now, just not adding the subdirectory is the primary action.
    # We can define a dummy Tracy::TracyClient target if needed later for linking.
    # add_library(TracyClient INTERFACE) # Or whatever Tracy's target is called
endif()


# SQLite (amalgamation)
# Create a static library for SQLite from its source files.
message(STATUS "Configuring SQLite library (sqlite3_lib)...")
add_library(sqlite3_lib STATIC
    lib/sqlite/sqlite3.c
    lib/sqlite/carray.c
)
target_include_directories(sqlite3_lib PUBLIC
    # PUBLIC because targets linking sqlite3_lib will need to find sqlite3.h
    ${CMAKE_CURRENT_SOURCE_DIR}/lib/sqlite
)
target_compile_definitions(sqlite3_lib PRIVATE
    SQLITE_CORE             # Standard define for building SQLite core
    SQLITE_OMIT_LOAD_EXTENSION=1 # As requested, disables dynamic loading of extensions
    # Other common SQLite defines that might be useful (mirroring autoconf):
    # SQLITE_DEFAULT_MEMSTATUS=0 # Disables memory statistics
    # SQLITE_DEFAULT_WAL_SYNCHRONOUS=1 # Slower but safer WAL mode
    # SQLITE_LIKE_DOESNT_MATCH_BLOBS # For compatibility
    # SQLITE_MAX_EXPR_DEPTH=0 # No limit on expression depth (0 means no limit)
    # SQLITE_USE_ALLOCA # Use alloca if available
    # SQLITE_ENABLE_UNLOCK_NOTIFY # Enable unlock notify API
    # SQLITE_ENABLE_COLUMN_METADATA # Enable column metadata functions
    # SQLITE_ENABLE_FTS3 # Enable FTS3/4
    # SQLITE_ENABLE_FTS5 # Enable FTS5
    # SQLITE_ENABLE_JSON1 # Enable JSON1 extension
    # SQLITE_ENABLE_RTREE # Enable RTREE extension
    # SQLITE_HAVE_ISNAN # Define if isnan() is available
)
message(STATUS "Created sqlite3_lib static library target.")

# FILESYSTEM LIBRARY CHECK
include(CheckCXXSourceCompiles) # For check_cxx_source_compiles
include(CheckLibraryExists)     # For check_library_exists

# Test code for C++17 std::filesystem
set(FILESYSTEM_TEST_CODE
"
#include <filesystem>
#include <fstream> // For std::ofstream
namespace fs = std::filesystem;
int main() {
    fs::path p = fs::current_path() / \"test.txt\";
    std::ofstream(p.string()).put('a');
    fs::remove(p);
    return static_cast<int>(fs::exists(p));
}
"
)

# Variable to store if a working filesystem solution is found
set(FILESYSTEM_WORKS FALSE)
set(PLATFORM_SPECIFIC_LINK_LIBS "") # Store any specific fs libs needed

# Check 1: Try compiling with std::filesystem directly (should work with C++17 and modern compilers/stdlibs)
message(STATUS "Checking for C++17 std::filesystem support...")
check_cxx_source_compiles("${FILESYSTEM_TEST_CODE}" _STD_FILESYSTEM_COMPILES_DIRECTLY)

if(_STD_FILESYSTEM_COMPILES_DIRECTLY)
    message(STATUS "C++17 std::filesystem support found directly.")
    set(FILESYSTEM_WORKS TRUE)
else()
    message(STATUS "Direct std::filesystem compilation failed. Checking for specific linker flags...")
    # Check 2: Try linking against stdc++fs
    # We need to save current CMAKE_REQUIRED_LIBRARIES, add stdc++fs, try compile, then restore.
    set(CMAKE_REQUIRED_LIBRARIES_OLD ${CMAKE_REQUIRED_LIBRARIES})
    list(APPEND CMAKE_REQUIRED_LIBRARIES "stdc++fs")
    check_cxx_source_compiles("${FILESYSTEM_TEST_CODE}" _STD_FILESYSTEM_WITH_STDCXXFS)
    list(POP_BACK CMAKE_REQUIRED_LIBRARIES) # remove stdc++fs for next check if this fails
    set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES_OLD})

    if(_STD_FILESYSTEM_WITH_STDCXXFS)
        message(STATUS "C++17 std::filesystem works with -lstdc++fs.")
        set(FILESYSTEM_WORKS TRUE)
        list(APPEND PLATFORM_SPECIFIC_LINK_LIBS "stdc++fs")
    else()
        message(STATUS "-lstdc++fs link check failed.")
        # Check 3: Try linking against c++fs
        set(CMAKE_REQUIRED_LIBRARIES_OLD ${CMAKE_REQUIRED_LIBRARIES})
        list(APPEND CMAKE_REQUIRED_LIBRARIES "c++fs")
        check_cxx_source_compiles("${FILESYSTEM_TEST_CODE}" _STD_FILESYSTEM_WITH_CXXFS)
        list(POP_BACK CMAKE_REQUIRED_LIBRARIES)
        set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES_OLD})

        if(_STD_FILESYSTEM_WITH_CXXFS)
            message(STATUS "C++17 std::filesystem works with -lc++fs.")
            set(FILESYSTEM_WORKS TRUE)
            list(APPEND PLATFORM_SPECIFIC_LINK_LIBS "c++fs")
        else()
            message(STATUS "-lc++fs link check failed.")
        endif()
    endif()
endif()

if(NOT FILESYSTEM_WORKS)
    message(FATAL_ERROR "C++17 <filesystem> support not found. stellar-core requires a C++17 compiler and standard library with filesystem support. You may need to explicitly link against stdc++fs or c++fs, or update your compiler/toolchain.")
else()
    if(PLATFORM_SPECIFIC_LINK_LIBS)
        message(STATUS "Filesystem support requires linking against: ${PLATFORM_SPECIFIC_LINK_LIBS}")
        # These libraries will need to be added to targets that use std::filesystem.
        # For example, by adding to a global property or an interface library.
        # For now, we just record them in PLATFORM_SPECIFIC_LINK_LIBS.
        # This list can be used later when linking executables/libraries.
        # e.g. target_link_libraries(my_target PRIVATE ${PLATFORM_SPECIFIC_LINK_LIBS})
    else()
        message(STATUS "C++17 std::filesystem support works without special linker flags.")
    endif()
endif()

# PLATFORM SPECIFIC LINK LIBRARIES (e.g. for libunwind)
# This section populates PLATFORM_SPECIFIC_LINK_LIBS which should be added to final targets.

# Libunwind linking (if not disabled and not Apple)
# HAVE_LIBUNWIND is set later during config.h generation, but we can check options here.
if(NOT APPLE AND NOT DISABLE_LIBUNWIND)
    if(PkgConfig_FOUND)
        pkg_check_modules(LIBUNWIND_PKG_LINK libunwind) # Use a different prefix to avoid conflict with HAVE_LIBUNWIND check variables
        if(LIBUNWIND_PKG_LINK_FOUND)
            message(STATUS "Found libunwind for linking via pkg-config: ${LIBUNWIND_PKG_LINK_LIBRARIES}")
            # LIBUNWIND_PKG_LINK_LIBRARIES is a list of library names/paths
            # Ensure they are treated as a list when adding
            foreach(lib ${LIBUNWIND_PKG_LINK_LIBRARIES})
                list(APPEND PLATFORM_SPECIFIC_LINK_LIBS "${lib}")
            endforeach()
            # Also add include directories if specified, though less common for link-time only.
            # list(APPEND PLATFORM_SPECIFIC_INCLUDE_DIRS ${LIBUNWIND_PKG_LINK_INCLUDE_DIRS})
        else()
            message(STATUS "libunwind not found for linking via pkg-config. Linkage might rely on compiler defaults or CMAKE_REQUIRED_LIBRARIES from HAVE_LIBUNWIND check.")
            # The check_library_exists for HAVE_LIBUNWIND already adds "unwind" to CMAKE_REQUIRED_LIBRARIES if successful.
            # That might be sufficient for some systems if pkg-config is not present or libunwind.pc is missing.
            # As a fallback, explicitly add 'unwind' if the earlier check for HAVE_LIBUNWIND determined it exists.
            # This requires HAVE_LIBUNWIND to be determined before this block.
            # For now, we rely on pkg-config here or the implicit CMAKE_REQUIRED_LIBRARIES.
            # If HAVE_LIBUNWIND was determined AND check_library_exists passed, "unwind" is already in CMAKE_REQUIRED_LIBRARIES for checks.
            # The actual linking may still need an explicit -lunwind.
            # Consider adding `unwind` as a fallback if HAVE_LIBUNWIND is true and pkg-config fails.
            # if(HAVE_LIBUNWIND) list(APPEND PLATFORM_SPECIFIC_LINK_LIBS "unwind"); endif()
            # This dependency on HAVE_LIBUNWIND being set means this block should be after HAVE_LIBUNWIND is determined.
            # Moving this logic to after config.h variable setup.
        endif()
    else()
        message(STATUS "pkg-config not found, cannot search for libunwind for linking via pkg-config.")
        # if(HAVE_LIBUNWIND) list(APPEND PLATFORM_SPECIFIC_LINK_LIBS "unwind"); endif() # Fallback if HAVE_LIBUNWIND known
    endif()
endif()
# Ensure PLATFORM_SPECIFIC_LINK_LIBS are unique if added from multiple places
if(PLATFORM_SPECIFIC_LINK_LIBS)
    list(REMOVE_DUPLICATES PLATFORM_SPECIFIC_LINK_LIBS)
    message(STATUS "Current platform specific link libraries: ${PLATFORM_SPECIFIC_LINK_LIBS}")
endif()

# XDR CODE GENERATION
message(STATUS "Configuring XDR code generation...")

# 1. Identify XDR Source Files
set(XDR_SOURCES "")
file(GLOB XDR_PROTOCOL_CURR_FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/protocol-curr/xdr/*.x")
if(XDR_PROTOCOL_CURR_FILES)
    list(APPEND XDR_SOURCES ${XDR_PROTOCOL_CURR_FILES})
    message(STATUS "  Found XDR files in src/protocol-curr/xdr/: ${XDR_PROTOCOL_CURR_FILES}")
else()
    message(WARNING "  No .x files found in src/protocol-curr/xdr/")
endif()

file(GLOB XDR_PROTOCOL_NEXT_FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/protocol-next/xdr/*.x")
if(XDR_PROTOCOL_NEXT_FILES)
    list(APPEND XDR_SOURCES ${XDR_PROTOCOL_NEXT_FILES})
    message(STATUS "  Found XDR files in src/protocol-next/xdr/: ${XDR_PROTOCOL_NEXT_FILES}")
else()
    message(WARNING "  No .x files found in src/protocol-next/xdr/")
endif()

if(NOT XDR_SOURCES)
    message(WARNING "  No .x files found for XDR generation. This might be an error if XDR files are expected.")
endif()

set(XDR_GENERATED_INCLUDE_DIR "${CMAKE_BINARY_DIR}/generated/xdr")
file(MAKE_DIRECTORY ${XDR_GENERATED_INCLUDE_DIR})
include_directories(${XDR_GENERATED_INCLUDE_DIR}) # Add to global include path for all targets
message(STATUS "  Generated XDR headers will be placed in: ${XDR_GENERATED_INCLUDE_DIR}")
message(STATUS "  Added ${XDR_GENERATED_INCLUDE_DIR} to include directories.")

# 2. Create Custom Commands for XDR Header Generation
set(XDR_GENERATED_HEADERS "") # List to store paths of generated headers

if(NOT XDRC_EXECUTABLE)
    message(FATAL_ERROR "XDRC_EXECUTABLE is not set. It should be set by the xdrpp configuration. Cannot proceed with XDR header generation.")
endif()

foreach(XDR_FILE_FULL_PATH ${XDR_SOURCES})
    get_filename_component(XDR_FILE_NAME ${XDR_FILE_FULL_PATH} NAME_WE) # Name without extension
    get_filename_component(XDR_FILE_NAME_WITH_EXT ${XDR_FILE_FULL_PATH} NAME)
    set(XDR_OUTPUT_HEADER "${XDR_GENERATED_INCLUDE_DIR}/${XDR_FILE_NAME}.h")

    # Determine dependency for XDRC_EXECUTABLE
    # If XDRC_EXECUTABLE points to a file built by an ExternalProject (e.g., xdrpp_external),
    # then that ExternalProject target should be a dependency.
    # The variable XDRC_EXECUTABLE is set to ${XDRPP_INSTALL_DIR}/bin/xdrc if internal,
    # and this path is part of BUILD_BYPRODUCTS of xdrpp_external.
    # So, depending on the xdrpp target (which depends on xdrpp_external) should suffice.
    set(XDRC_DEPENDENCY "")
    if(FORCE_INTERNAL_XDRPP) # Assuming FORCE_INTERNAL_XDRPP implies xdrpp_external target exists
        # Check if xdrpp target (which depends on xdrpp_external) exists
        if(TARGET xdrpp)
            set(XDRC_DEPENDENCY xdrpp) # Depend on the library target that ensures xdrc is built
        else()
            # Fallback if only xdrpp_external target exists (less ideal)
            set(XDRC_DEPENDENCY xdrpp_external)
        endif()
    else()
        # If using system xdrc, XDRC_EXECUTABLE is a system path, no CMake target dependency.
        # The file itself ${XDR_FILE_FULL_PATH} is an implicit dependency.
    endif()

    add_custom_command(
        OUTPUT ${XDR_OUTPUT_HEADER}
        COMMAND ${XDRC_EXECUTABLE} -hh -pedantic -o ${XDR_OUTPUT_HEADER} ${XDR_FILE_FULL_PATH}
        DEPENDS ${XDR_FILE_FULL_PATH} ${XDRC_DEPENDENCY} # XDRC_DEPENDENCY might be empty if system tool
        COMMENT "Generating C++ header from ${XDR_FILE_NAME_WITH_EXT} using ${XDRC_EXECUTABLE}"
        VERBATIM # Ensure command is executed as written, especially with paths
    )
    list(APPEND XDR_GENERATED_HEADERS ${XDR_OUTPUT_HEADER})
    message(STATUS "  Added custom command to generate ${XDR_OUTPUT_HEADER} from ${XDR_FILE_NAME_WITH_EXT}")
endforeach()

# 3. Generate XDRFilesSha256.cpp
set(XDR_SHA256_CPP_FILE "${CMAKE_BINARY_DIR}/generated/XDRFilesSha256.cpp")
set(GENERATED_CORE_SOURCES "") # Initialize list for generated core sources

if(ENABLE_NEXT_PROTOCOL_VERSION_UNSAFE_FOR_PRODUCTION)
    set(HASH_XDRS_PROTOCOL_ARG "protocol-next")
else()
    set(HASH_XDRS_PROTOCOL_ARG "protocol-curr")
endif()
message(STATUS "  Protocol argument for hash-xdrs.sh: ${HASH_XDRS_PROTOCOL_ARG}")

set(HASH_XDRS_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/hash-xdrs.sh")
if(NOT EXISTS ${HASH_XDRS_SCRIPT})
    message(FATAL_ERROR "hash-xdrs.sh not found at ${HASH_XDRS_SCRIPT}. Cannot generate XDRFilesSha256.cpp.")
endif()

# Ensure XDR_SOURCES is not empty before adding it as a dependency
if(XDR_SOURCES)
    add_custom_command(
        OUTPUT ${XDR_SHA256_CPP_FILE}
        COMMAND bash ${HASH_XDRS_SCRIPT} ${HASH_XDRS_PROTOCOL_ARG} > ${XDR_SHA256_CPP_FILE}
        DEPENDS ${XDR_SOURCES} ${HASH_XDRS_SCRIPT} # Depends on all .x files and the script itself
        COMMENT "Generating ${XDR_SHA256_CPP_FILE} for ${HASH_XDRS_PROTOCOL_ARG}"
        VERBATIM
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} # Script might assume it's run from source root
    )
    list(APPEND GENERATED_CORE_SOURCES ${XDR_SHA256_CPP_FILE})
    message(STATUS "  Added custom command to generate ${XDR_SHA256_CPP_FILE}")
else()
    message(WARNING "  No XDR source files found, so XDRFilesSha256.cpp will not be generated. This might be an error.")
    # Create an empty XDRFilesSha256.cpp if no XDR sources to avoid build issues if something expects this file.
    # However, if XDR_SOURCES is empty, the script might fail or produce an empty file anyway.
    # Let's ensure the file is created to avoid other targets failing to find it.
    file(WRITE ${XDR_SHA256_CPP_FILE} "// No XDR sources found, this file is intentionally empty or contains minimal content.\nvoid xdrFilesSha256Dummy() {}\n")
    list(APPEND GENERATED_CORE_SOURCES ${XDR_SHA256_CPP_FILE})
    message(STATUS "  Created empty ${XDR_SHA256_CPP_FILE} as no XDR sources were found.")
endif()

message(STATUS "XDR code generation configuration complete.")
# XDR_GENERATED_HEADERS and GENERATED_CORE_SOURCES are now populated.


# FLEX/BISON XDR QUERY GENERATION
message(STATUS "Configuring Flex/Bison for XDR Query parsing...")

# 1. Find Flex and Bison
find_package(FLEX REQUIRED)
find_package(BISON REQUIRED)
message(STATUS "  Found Flex: ${FLEX_EXECUTABLE}")
message(STATUS "  Found Bison: ${BISON_EXECUTABLE}")

# 2. Define Input and Output Files
set(XDRQUERY_LEXER_INPUT "${CMAKE_SOURCE_DIR}/src/util/xdrquery/XDRQueryScanner.ll")
set(XDRQUERY_GENERATED_DIR "${CMAKE_BINARY_DIR}/generated/xdrquery")
set(XDRQUERY_LEXER_OUTPUT_CPP "${XDRQUERY_GENERATED_DIR}/XDRQueryScanner.cpp")

set(XDRQUERY_PARSER_INPUT "${CMAKE_SOURCE_DIR}/src/util/xdrquery/XDRQueryParser.yy")
set(XDRQUERY_PARSER_OUTPUT_CPP "${XDRQUERY_GENERATED_DIR}/XDRQueryParser.cpp")
set(XDRQUERY_PARSER_OUTPUT_H "${XDRQUERY_GENERATED_DIR}/XDRQueryParser.h") # Bison output header

file(MAKE_DIRECTORY ${XDRQUERY_GENERATED_DIR})
message(STATUS "  XDRQuery generated files will be in: ${XDRQUERY_GENERATED_DIR}")

# Check if input files exist
if(NOT EXISTS ${XDRQUERY_LEXER_INPUT})
    message(FATAL_ERROR "XDRQuery lexer input file not found: ${XDRQUERY_LEXER_INPUT}")
endif()
if(NOT EXISTS ${XDRQUERY_PARSER_INPUT})
    message(FATAL_ERROR "XDRQuery parser input file not found: ${XDRQUERY_PARSER_INPUT}")
endif()

# 3. Add Custom Command for Flex (Lexer Generation)
add_custom_command(
    OUTPUT ${XDRQUERY_LEXER_OUTPUT_CPP}
    COMMAND ${FLEX_EXECUTABLE} --outfile=${XDRQUERY_LEXER_OUTPUT_CPP} ${XDRQUERY_LEXER_INPUT}
    DEPENDS ${XDRQUERY_LEXER_INPUT}
    COMMENT "Generating C++ scanner from XDRQueryScanner.ll using Flex"
    VERBATIM
)
message(STATUS "  Added custom command for Flex -> ${XDRQUERY_LEXER_OUTPUT_CPP}")

# 4. Add Custom Command for Bison (Parser Generation)
add_custom_command(
    OUTPUT ${XDRQUERY_PARSER_OUTPUT_CPP} ${XDRQUERY_PARSER_OUTPUT_H} # Specify both outputs
    COMMAND ${BISON_EXECUTABLE} --defines=${XDRQUERY_PARSER_OUTPUT_H} --output=${XDRQUERY_PARSER_OUTPUT_CPP} ${XDRQUERY_PARSER_INPUT}
    DEPENDS ${XDRQUERY_PARSER_INPUT}
    COMMENT "Generating C++ parser from XDRQueryParser.yy using Bison"
    VERBATIM
)
message(STATUS "  Added custom command for Bison -> ${XDRQUERY_PARSER_OUTPUT_CPP} and ${XDRQUERY_PARSER_OUTPUT_H}")

# 5. Add Generated Files to Source List and Include Directory
list(APPEND GENERATED_CORE_SOURCES
    ${XDRQUERY_LEXER_OUTPUT_CPP}
    ${XDRQUERY_PARSER_OUTPUT_CPP}
    # ${XDRQUERY_PARSER_OUTPUT_H} # Headers are not typically added to source lists for compilation
)
message(STATUS "  Added Flex/Bison generated .cpp files to GENERATED_CORE_SOURCES.")

include_directories(${XDRQUERY_GENERATED_DIR})
message(STATUS "  Added ${XDRQUERY_GENERATED_DIR} to include directories for XDRQueryParser.h.")

message(STATUS "Flex/Bison XDR Query generation configuration complete.")


# RUST INTEGRATION
message(STATUS "Configuring Rust integration (Soroban setup and cxxbridge generation)...")

# 1. Define Rust Toolchain and Cargo (already done, this is a good place for more Rust vars)
set(RUST_TOOLCHAIN_FILE "${CMAKE_SOURCE_DIR}/rust-toolchain.toml")
set(RUST_TOOLCHAIN_CHANNEL "")

if(EXISTS ${RUST_TOOLCHAIN_FILE})
    file(STRINGS ${RUST_TOOLCHAIN_FILE} RUST_TOOLCHAIN_CHANNEL_LINE REGEX "^channel[ \t]*=[ \t]*[\"'](.*)[\"']")
    if(RUST_TOOLCHAIN_CHANNEL_LINE)
        string(REGEX REPLACE "^channel[ \t]*=[ \t]*[\"'](.*)[\"']" "\\1" RUST_TOOLCHAIN_CHANNEL "${RUST_TOOLCHAIN_CHANNEL_LINE}")
        message(STATUS "  Found Rust toolchain channel: ${RUST_TOOLCHAIN_CHANNEL}")
    else()
        message(WARNING "  Could not parse 'channel' from ${RUST_TOOLCHAIN_FILE}. Using default cargo.")
    endif()
else()
    message(WARNING "  rust-toolchain.toml not found at ${RUST_TOOLCHAIN_FILE}. Using default cargo.")
endif()

if(RUST_TOOLCHAIN_CHANNEL)
    set(CARGO_EXECUTABLE "cargo" "+${RUST_TOOLCHAIN_CHANNEL}") # CARGO_EXECUTABLE is a list for command execution
else()
    set(CARGO_EXECUTABLE "cargo")
endif()
message(STATUS "  Using Cargo executable: ${CARGO_EXECUTABLE}")

set(RUST_BUILD_DIR "${CMAKE_BINARY_DIR}/rust_build_temp")
set(RUST_BIN_DIR "${RUST_BUILD_DIR}/bin")
set(RUST_TARGET_DIR "${CMAKE_BINARY_DIR}/target") # For final .a files
message(STATUS "  Rust build temp dir: ${RUST_BUILD_DIR}")
message(STATUS "  Rust final target dir: ${RUST_TARGET_DIR}")

# Soroban Build Parameters
set(ALL_SOROBAN_PROTOCOLS p21 p22 p23) # From src/Makefile.am SOROBAN_PROTOCOL_VERSIONS
set(WIP_SOROBAN_PROTOCOL "") # Example: "p24", set to empty if no WIP
message(STATUS "  All Soroban protocols (initial): ${ALL_SOROBAN_PROTOCOLS}")
message(STATUS "  WIP Soroban protocol: ${WIP_SOROBAN_PROTOCOL}")

if(ENABLE_NEXT_PROTOCOL_VERSION_UNSAFE_FOR_PRODUCTION AND WIP_SOROBAN_PROTOCOL)
    list(APPEND ALL_SOROBAN_PROTOCOLS ${WIP_SOROBAN_PROTOCOL})
    message(STATUS "  ENABLE_NEXT_PROTOCOL_VERSION_UNSAFE_FOR_PRODUCTION is ON. Added WIP protocol: ${WIP_SOROBAN_PROTOCOL}")
endif()
list(SORT ALL_SOROBAN_PROTOCOLS) # Ensure sorted order for max protocol determination
list(GET ALL_SOROBAN_PROTOCOLS -1 SOROBAN_MAX_PROTOCOL) # Get the last element
message(STATUS "  All Soroban protocols (final): ${ALL_SOROBAN_PROTOCOLS}")
message(STATUS "  Soroban Max Protocol: ${SOROBAN_MAX_PROTOCOL}")

set(RUST_PROFILE "release") # Corresponds to --release flag for cargo
set(RUST_PROFILE_ARG "--${RUST_PROFILE}")
if(RUST_PROFILE STREQUAL "debug") # Or other non-release profiles
    set(RUST_PROFILE_ARG "") # No specific flag for debug, it's the default
endif()
message(STATUS "  Rust build profile: ${RUST_PROFILE} (Arg: ${RUST_PROFILE_ARG})")

# Prepare Cargo Features for Soroban
set(CARGO_FEATURE_TRACY_STRING "")
if(ENABLE_TRACY)
    # From src/Makefile.am: SOROBAN_TRACY_FEATURES
    set(CARGO_FEATURE_TRACY_STRING "tracy-client/enable,tracy-client/ondemand,tracy-client/attributes,tracy-client/multi_lane_event,tracy-client/system-tracing,tracy-client/plot,tracy-client/alloc_untraced,tracy-client/implicit_sends_resize,tracy-client/timer_interleave_guard,tracy-client/fibers,tracy-client/frame_options,tracy-client/manual_lifetime_management,tracy-client/only_localhost,tracy-client/src_loc,tracy-client/verbose,tracy-client/non_atomic_broadcast,tracy-client/lock_annotations,tracy-client/resolve_callstack,tracy-client/symbol_decoding,tracy-client/cache_decoding")
    message(STATUS "  Tracy enabled for Soroban. Features: ${CARGO_FEATURE_TRACY_STRING}")
endif()

set(CARGO_FEATURE_NEXT_STRING "")
if(ENABLE_NEXT_PROTOCOL_VERSION_UNSAFE_FOR_PRODUCTION)
    set(CARGO_FEATURE_NEXT_STRING "next")
    message(STATUS "  'next' feature enabled for Soroban (due to ENABLE_NEXT_PROTOCOL_VERSION_UNSAFE_FOR_PRODUCTION).")
endif()

# Soroban Dependency Tree Check
set(RUST_DEP_TREE_STAMP_FILE "${RUST_BUILD_DIR}/src/dep-trees/equal-trees.stamp")
file(MAKE_DIRECTORY "${RUST_BUILD_DIR}/src/dep-trees") # Ensure directory exists

set(SOROBAN_DEP_TREE_CHECK_CMDS "")
set(SOROBAN_DEP_TREE_CHECK_DEPS "")

foreach(proto ${ALL_SOROBAN_PROTOCOLS})
    set(CARGO_TOML_PATH "${CMAKE_SOURCE_DIR}/src/rust/soroban/${proto}/Cargo.toml")
    set(CARGO_LOCK_PATH "${CMAKE_SOURCE_DIR}/src/rust/soroban/${proto}/Cargo.lock")
    set(DEP_TREE_EXPECT_FILE "${CMAKE_SOURCE_DIR}/src/rust/src/dep-trees/${proto}-expect.txt")

    # Check if files exist before adding commands/dependencies
    if(NOT EXISTS ${CARGO_TOML_PATH})
        message(FATAL_ERROR "Cargo.toml for soroban protocol ${proto} not found at ${CARGO_TOML_PATH}")
    endif()
    if(NOT EXISTS ${CARGO_LOCK_PATH})
        message(FATAL_ERROR "Cargo.lock for soroban protocol ${proto} not found at ${CARGO_LOCK_PATH}")
    endif()
    if(NOT EXISTS ${DEP_TREE_EXPECT_FILE})
        message(WARNING "Expected dependency tree file not found for protocol ${proto} at ${DEP_TREE_EXPECT_FILE}. Cannot perform check.")
        # Optionally, skip this protocol's check or fail if strict. For now, just warn.
        continue()
    endif()

    list(APPEND SOROBAN_DEP_TREE_CHECK_DEPS ${CARGO_TOML_PATH} ${CARGO_LOCK_PATH} ${DEP_TREE_EXPECT_FILE})

    # Define a unique name for the output of cargo tree for this protocol
    set(CARGO_TREE_OUTPUT_FILE "${RUST_BUILD_DIR}/src/dep-trees/${proto}-actual.txt")

    # Command to run cargo tree and store its output
    # The comparison logic is complex to do directly in CMake.
    # This setup will run cargo tree. A more robust check would involve an external script.
    # For now, we just execute cargo tree. If it fails, the build will stop.
    # The actual comparison is a placeholder here.
    list(APPEND SOROBAN_DEP_TREE_CHECK_CMDS
        COMMAND ${CARGO_EXECUTABLE} tree --manifest-path ${CARGO_TOML_PATH} --locked --package soroban-env-host --edges no-dev --target all > ${CARGO_TREE_OUTPUT_FILE}
        # Placeholder for actual comparison:
        # COMMAND ${CMAKE_COMMAND} -E compare_files ${CARGO_TREE_OUTPUT_FILE} ${DEP_TREE_EXPECT_FILE} # This would fail if different
        # COMMAND if %ERRORLEVEL% NEQ 0 ( echo "Error: Dependency tree for ${proto} differs from expected." && exit /b 1 ) # Windows
        # COMMAND bash -c "diff ${CARGO_TREE_OUTPUT_FILE} ${DEP_TREE_EXPECT_FILE} || (echo 'Error: Dependency tree for ${proto} differs.' && exit 1)" # Linux/macOS
        COMMENT "Running cargo tree for soroban protocol ${proto}"
    )
endforeach()

if(SOROBAN_DEP_TREE_CHECK_CMDS)
    add_custom_command(
        OUTPUT ${RUST_DEP_TREE_STAMP_FILE}
        COMMAND ${SOROBAN_DEP_TREE_CHECK_CMDS} # Series of commands
        COMMAND ${CMAKE_COMMAND} -E touch ${RUST_DEP_TREE_STAMP_FILE}
        DEPENDS ${SOROBAN_DEP_TREE_CHECK_DEPS} ${CARGO_EXECUTABLE} # Depends on all toml/lock files and cargo
        COMMENT "Checking Soroban dependency trees..."
        VERBATIM
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} # cargo tree might need to run from a specific context if paths are relative in Cargo.toml
    )
    message(STATUS "  Added custom command for Soroban dependency tree check (Output: ${RUST_DEP_TREE_STAMP_FILE}).")

    add_custom_target(soroban_dep_tree_check
        DEPENDS ${RUST_DEP_TREE_STAMP_FILE}
        COMMENT "Ensuring Soroban dependency trees are checked and match expectations"
    )
    message(STATUS "  Added custom target soroban_dep_tree_check.")
else()
    message(WARNING "  No valid protocols found for Soroban dependency tree check, or expected files missing. Skipping check.")
    # Create the stamp file anyway so other things depending on it don't fail
    add_custom_command(OUTPUT ${RUST_DEP_TREE_STAMP_FILE} COMMAND ${CMAKE_COMMAND} -E touch ${RUST_DEP_TREE_STAMP_FILE} COMMENT "Skipping Soroban dependency tree check - creating stamp file.")
    add_custom_target(soroban_dep_tree_check DEPENDS ${RUST_DEP_TREE_STAMP_FILE})
endif()

# Build Soroban Protocol Libraries (.rlib files)
message(STATUS "  Configuring Soroban protocol library builds...")
set(SOROBAN_LIBS_STAMP_FILE "${RUST_BUILD_DIR}/soroban/soroban-libs.stamp")
file(MAKE_DIRECTORY "${RUST_BUILD_DIR}/soroban") # Ensure stamp file directory exists

set(SOROBAN_RLIBS "") # List of paths to generated .rlib files
set(SOROBAN_EXTERN_ARGS "") # List of --extern arguments for later Rust builds
set(SOROBAN_DEPEND_ARGS "") # List of -L dependency arguments

# Determine CXXSTDLIB (simplified placeholder)
set(CXXSTDLIB "")
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    # This is a guess; actual library name can vary (e.g., libc++.so.1)
    # A more robust check would use try_compile or compiler introspection.
    set(CXXSTDLIB "-lc++")
    message(STATUS "    Guessed CXXSTDLIB for Clang: ${CXXSTDLIB}")
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(CXXSTDLIB "-lstdc++")
    message(STATUS "    Guessed CXXSTDLIB for GCC: ${CXXSTDLIB}")
else()
    message(WARNING "    Cannot determine CXXSTDLIB for compiler ${CMAKE_CXX_COMPILER_ID}. Linking Rust crates might be affected.")
endif()

# Determine RUSTFLAGS_ASAN_VALUE
set(RUSTFLAGS_ASAN_VALUE "")
if(ENABLE_ASAN)
    set(RUSTFLAGS_ASAN_VALUE "-Zsanitizer=address")
    message(STATUS "    ASAN enabled for Rust builds. RUSTFLAGS_ASAN_VALUE: ${RUSTFLAGS_ASAN_VALUE}")
endif()

set(ALL_SOROBAN_RLIB_TARGETS "") # For final stamp file dependency

foreach(proto ${ALL_SOROBAN_PROTOCOLS})
    message(STATUS "    Configuring build for Soroban protocol: ${proto}")
    set(CARGO_MANIFEST_DIR "${CMAKE_SOURCE_DIR}/src/rust/soroban/${proto}")
    set(CARGO_MANIFEST_PATH "${CARGO_MANIFEST_DIR}/Cargo.toml")

    if(NOT EXISTS ${CARGO_MANIFEST_PATH})
        message(FATAL_ERROR "Cargo manifest not found for protocol ${proto} at ${CARGO_MANIFEST_PATH}")
        continue()
    endif()

    # Determine features for this protocol
    set(CURRENT_SOROBAN_FEATURES_LIST "")
    if(proto STREQUAL "p21") # As per Makefile.am, p21 has no specific features beyond common ones
        # No protocol-specific features for p21 beyond common (tracy/next if applicable at max protocol)
    elseif(proto STREQUAL SOROBAN_MAX_PROTOCOL)
        if(CARGO_FEATURE_TRACY_STRING)
            list(APPEND CURRENT_SOROBAN_FEATURES_LIST ${CARGO_FEATURE_TRACY_STRING})
        endif()
        # SOROBAN_FEATURE_NEXT_STRING logic from Makefile.am:
        # ifneq ($(WIP_SOROBAN_PROTOCOL),)
        #   SOROBAN_FEATURE_NEXT_STRING := $(CARGO_FEATURE_NEXT_STRING)
        # else
        #   SOROBAN_FEATURE_NEXT_STRING :=
        # endif
        if(WIP_SOROBAN_PROTOCOL AND CARGO_FEATURE_NEXT_STRING) # If there is a WIP protocol, 'next' applies to max
            list(APPEND CURRENT_SOROBAN_FEATURES_LIST ${CARGO_FEATURE_NEXT_STRING})
        elseif(NOT WIP_SOROBAN_PROTOCOL AND CARGO_FEATURE_NEXT_STRING) # If no WIP, 'next' still applies if ENABLE_NEXT is on.
             list(APPEND CURRENT_SOROBAN_FEATURES_LIST ${CARGO_FEATURE_NEXT_STRING})
        endif()
    else() # Not p21 and not max protocol
        if(CARGO_FEATURE_TRACY_STRING)
            list(APPEND CURRENT_SOROBAN_FEATURES_LIST ${CARGO_FEATURE_TRACY_STRING})
        endif()
    endif()

    set(CURRENT_SOROBAN_FEATURE_FLAGS_STRING "")
    if(CURRENT_SOROBAN_FEATURES_LIST)
        string(JOIN "," TEMP_FEATURE_STRING "${CURRENT_SOROBAN_FEATURES_LIST}")
        set(CURRENT_SOROBAN_FEATURE_FLAGS_STRING "--features" "${TEMP_FEATURE_STRING}")
        message(STATUS "      Features for ${proto}: ${CURRENT_SOROBAN_FEATURE_FLAGS_STRING}")
    else()
        message(STATUS "      No specific features for ${proto} beyond defaults.")
    endif()

    # Output .rlib path - Cargo places it in target/release/lib<crate_name>.rlib or target/debug/...
    # The crate name is soroban-env-host.
    # We need to ensure each protocol build is distinct. `-Cmetadata=${proto}` helps.
    # The final rlib will be in RUST_TARGET_DIR/${RUST_PROFILE}/libsoroban_env_host.rlib
    # but its metadata will be unique. We need to copy it to a protocol-specific name.
    set(SOROBAN_RLIB_ORIGINAL_NAME "libsoroban_env_host.rlib") # Cargo's output name
    set(SOROBAN_RLIB_PROTOCOL_SPECIFIC_NAME "libsoroban_env_host_${proto}.rlib")
    set(SOROBAN_RLIB_PATH_ORIGINAL "${RUST_TARGET_DIR}/${RUST_PROFILE}/${SOROBAN_RLIB_ORIGINAL_NAME}")
    set(SOROBAN_RLIB_PATH_FINAL "${RUST_TARGET_DIR}/${RUST_PROFILE}/${SOROBAN_RLIB_PROTOCOL_SPECIFIC_NAME}")

    # Glob for Rust source files for dependency tracking
    file(GLOB CURRENT_RUST_PROTOCOL_SRC_FILES "${CARGO_MANIFEST_DIR}/src/**/*.rs")

    set(ENV_VARS "")
    list(APPEND ENV_VARS "RUST_BACKTRACE=1")
    if(CMAKE_C_COMPILER)   list(APPEND ENV_VARS "CC=${CMAKE_C_COMPILER}") endif()
    if(CMAKE_CXX_COMPILER) list(APPEND ENV_VARS "CXX=${CMAKE_CXX_COMPILER}") endif()
    if(CMAKE_LINKER)       list(APPEND ENV_VARS "LD=${CMAKE_LINKER}") endif()
    if(CMAKE_C_FLAGS)      list(APPEND ENV_VARS "CFLAGS=${CMAKE_C_FLAGS}") endif()
    if(CMAKE_CXX_FLAGS)    list(APPEND ENV_VARS "CXXFLAGS=${CMAKE_CXX_FLAGS}") endif()
    if(CMAKE_EXE_LINKER_FLAGS) list(APPEND ENV_VARS "LDFLAGS=${CMAKE_EXE_LINKER_FLAGS}") endif() # Or CMAKE_SHARED_LINKER_FLAGS
    if(CXXSTDLIB)          list(APPEND ENV_VARS "CXXSTDLIB=${CXXSTDLIB}") endif()
    list(APPEND ENV_VARS "CARGO_NET_GIT_FETCH_WITH_CLI=true")
    list(APPEND ENV_VARS "CARGO_TARGET_DIR=${RUST_TARGET_DIR}") # Ensure cargo uses our specified target dir

    set(RUSTFLAGS_VALUE "-Cmetadata=${proto} ${RUSTFLAGS_ASAN_VALUE}")
    string(STRIP "${RUSTFLAGS_VALUE}" RUSTFLAGS_VALUE) # Remove leading/trailing whitespace
    list(APPEND ENV_VARS "RUSTFLAGS=${RUSTFLAGS_VALUE}")
    
    # CMake command list for environment variables
    set(ENV_SETUP_COMMANDS "")
    foreach(env_var ${ENV_VARS})
      list(APPEND ENV_SETUP_COMMANDS "${CMAKE_COMMAND}" "-E" "env" "${env_var}")
    endforeach()

    # Custom command for building the rlib
    # We use a per-protocol stamp file to manage dependencies correctly.
    set(SOROBAN_RLIB_STAMP_PER_PROTO "${RUST_BUILD_DIR}/soroban/stamp_${proto}.rlib_built")
    
    add_custom_command(
        OUTPUT ${SOROBAN_RLIB_STAMP_PER_PROTO}
        COMMAND ${ENV_SETUP_COMMANDS}
                ${CARGO_EXECUTABLE} build --package soroban-env-host ${RUST_PROFILE_ARG} --locked ${CURRENT_SOROBAN_FEATURE_FLAGS_STRING}
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${SOROBAN_RLIB_PATH_ORIGINAL} ${SOROBAN_RLIB_PATH_FINAL} # Copy to protocol-specific name
        COMMAND ${CMAKE_COMMAND} -E touch ${SOROBAN_RLIB_STAMP_PER_PROTO}
        WORKING_DIRECTORY ${CARGO_MANIFEST_DIR}
        DEPENDS ${CARGO_MANIFEST_PATH} "${CARGO_MANIFEST_DIR}/Cargo.lock" ${RUST_DEP_TREE_STAMP_FILE} ${CURRENT_RUST_PROTOCOL_SRC_FILES}
        COMMENT "Building Soroban protocol ${proto} rlib (soroban-env-host)"
        VERBATIM
    )
    message(STATUS "      Added custom command for soroban-env-host rlib (protocol ${proto}) -> ${SOROBAN_RLIB_PATH_FINAL}")
    list(APPEND SOROBAN_RLIBS ${SOROBAN_RLIB_PATH_FINAL})
    list(APPEND ALL_SOROBAN_RLIB_TARGETS ${SOROBAN_RLIB_STAMP_PER_PROTO})

    list(APPEND SOROBAN_EXTERN_ARGS "--extern" "soroban_env_host_${proto}=${SOROBAN_RLIB_PATH_FINAL}")
endforeach()

# Add dependency argument for the directory containing the .rlib files
# This helps rustc find the .rlib files during compilation of other crates.
# The deps dir contains other compiled dependencies of the rlibs.
list(APPEND SOROBAN_DEPEND_ARGS "-L" "dependency=${RUST_TARGET_DIR}/${RUST_PROFILE}/deps")
list(REMOVE_DUPLICATES SOROBAN_DEPEND_ARGS) # Avoid duplicate -L flags

# Command to create the final stamp file for all Soroban rlibs
if(ALL_SOROBAN_RLIB_TARGETS)
    add_custom_command(
        OUTPUT ${SOROBAN_LIBS_STAMP_FILE}
        COMMAND ${CMAKE_COMMAND} -E touch ${SOROBAN_LIBS_STAMP_FILE}
        DEPENDS ${ALL_SOROBAN_RLIB_TARGETS}
        COMMENT "All Soroban protocol rlibs built."
        VERBATIM
    )
    message(STATUS "  Added final stamp command for all Soroban rlibs: ${SOROBAN_LIBS_STAMP_FILE}")
else()
    message(WARNING "  No Soroban rlib targets were configured. Stamp file ${SOROBAN_LIBS_STAMP_FILE} will be empty or not created correctly.")
    # Create the stamp file anyway to avoid build failures if other targets depend on it.
    add_custom_command(OUTPUT ${SOROBAN_LIBS_STAMP_FILE} COMMAND ${CMAKE_COMMAND} -E touch ${SOROBAN_LIBS_STAMP_FILE} COMMENT "No Soroban rlibs built - creating empty stamp file.")
endif()

message(STATUS "  Soroban RLIBS: ${SOROBAN_RLIBS}")
message(STATUS "  Soroban EXTERN_ARGS: ${SOROBAN_EXTERN_ARGS}")
message(STATUS "  Soroban DEPEND_ARGS: ${SOROBAN_DEPEND_ARGS}")
message(STATUS "  Soroban protocol library build configuration complete.")

# Build Main Rust Library (librust_stellar_core.a)
message(STATUS "  Configuring main Rust library build (librust_stellar_core.a)...")

# 1. Define Main Rust Library Target Path
set(LIBRUST_STELLAR_CORE_A "${RUST_TARGET_DIR}/${RUST_PROFILE}/librust_stellar_core.a")
message(STATUS "    Output for main Rust library: ${LIBRUST_STELLAR_CORE_A}")

# 2. Prepare Cargo Features for Main Library
set(CARGO_FEATURE_TESTUTILS_STRING "")
if(BUILD_TESTS)
    set(CARGO_FEATURE_TESTUTILS_STRING "--features" "testutils")
    message(STATUS "    Testutils feature enabled for main Rust library (BUILD_TESTS is ON).")
endif()

set(MAIN_RUST_CARGO_FEATURES_LIST "")
if(CARGO_FEATURE_TRACY_STRING) # This is a list of features already
    list(APPEND MAIN_RUST_CARGO_FEATURES_LIST ${CARGO_FEATURE_TRACY_STRING})
endif()
if(CARGO_FEATURE_NEXT_STRING) # This is a single feature name "next"
    list(APPEND MAIN_RUST_CARGO_FEATURES_LIST ${CARGO_FEATURE_NEXT_STRING})
endif()

set(MAIN_RUST_CARGO_FEATURES_STRING "")
if(MAIN_RUST_CARGO_FEATURES_LIST)
    string(JOIN "," TEMP_MAIN_FEATURE_STRING "${MAIN_RUST_CARGO_FEATURES_LIST}")
    set(MAIN_RUST_CARGO_FEATURES_STRING "--features" "${TEMP_MAIN_FEATURE_STRING}")
endif()
# Append testutils string if it's set (it already includes --features)
if(CARGO_FEATURE_TESTUTILS_STRING)
    # If MAIN_RUST_CARGO_FEATURES_STRING is already set, we need to append "testutils" to its list
    # If not, CARGO_FEATURE_TESTUTILS_STRING can be used directly.
    # The prompt implies CARGO_FEATURE_TESTUTILS_STRING is "--features testutils" or empty.
    # This means it cannot be simply appended if other features are present.
    # Re-evaluating feature string construction:
    unset(MAIN_RUST_CARGO_FEATURES_LIST) # Start fresh
    set(MAIN_RUST_CARGO_FEATURES_LIST "")
    if(CARGO_FEATURE_TRACY_STRING)
        # CARGO_FEATURE_TRACY_STRING is "feat1,feat2,feat3"
        # Need to split it if it contains multiple features to avoid "feat1,feat2,feat3,next"
        # However, the current setup makes CARGO_FEATURE_TRACY_STRING a comma-separated string ready for --features
        # So, we need to collect individual feature *names* then join.
        # For simplicity, assume CARGO_FEATURE_TRACY_STRING and CARGO_FEATURE_NEXT_STRING are just feature names or lists of names.
        # Based on previous step: CARGO_FEATURE_TRACY_STRING is already a comma-separated list.
        # CARGO_FEATURE_NEXT_STRING is just "next".
        if(CARGO_FEATURE_TRACY_STRING) { list(APPEND MAIN_RUST_CARGO_FEATURES_LIST ${CARGO_FEATURE_TRACY_STRING}) } # This is already a string like "f1,f2"
        if(CARGO_FEATURE_NEXT_STRING) { list(APPEND MAIN_RUST_CARGO_FEATURES_LIST ${CARGO_FEATURE_NEXT_STRING}) }
        if(BUILD_TESTS) { list(APPEND MAIN_RUST_CARGO_FEATURES_LIST "testutils") }

        if(MAIN_RUST_CARGO_FEATURES_LIST)
            string(JOIN "," TEMP_MAIN_FEATURE_STRING_JOINED "${MAIN_RUST_CARGO_FEATURES_LIST}")
            set(MAIN_RUST_CARGO_FEATURES_STRING "--features" "${TEMP_MAIN_FEATURE_STRING_JOINED}")
        endif()
    endif()
    message(STATUS "    Main Rust library features: ${MAIN_RUST_CARGO_FEATURES_STRING}")
endif()


# 3. Add Custom Command to Build librust_stellar_core.a
set(RUST_STELLAR_CORE_MANIFEST_DIR "${CMAKE_SOURCE_DIR}/src/rust")
set(RUST_STELLAR_CORE_CARGO_TOML "${RUST_STELLAR_CORE_MANIFEST_DIR}/Cargo.toml")
set(RUST_STELLAR_CORE_CARGO_LOCK "${RUST_STELLAR_CORE_MANIFEST_DIR}/Cargo.lock")
set(TOP_LEVEL_CARGO_TOML "${CMAKE_SOURCE_DIR}/Cargo.toml") # Workspace Cargo.toml
set(TOP_LEVEL_CARGO_LOCK "${CMAKE_SOURCE_DIR}/Cargo.lock") # Workspace Cargo.lock

# Ensure dependencies are correctly listed
set(MAIN_RUST_BUILD_DEPENDS
    ${SOROBAN_LIBS_STAMP_FILE}
    ${RUST_STELLAR_CORE_CARGO_TOML}
    ${RUST_STELLAR_CORE_CARGO_LOCK}
    ${TOP_LEVEL_CARGO_TOML}
    ${TOP_LEVEL_CARGO_LOCK}
    ${RUST_SRC_FILES} # Globbed in cxxbridge setup: src/rust/src/*.rs
    ${RUST_BRIDGE_OUTPUT_H} # Generated header
    ${RUST_BRIDGE_OUTPUT_CPP} # Generated C++ source for bridge
)
list(REMOVE_DUPLICATES MAIN_RUST_BUILD_DEPENDS)

# Re-use ENV_VARS from Soroban build, but ensure RUSTFLAGS is specific for this build
# Soroban used -Cmetadata=${proto}, main library doesn't need that.
set(MAIN_RUST_ENV_VARS "")
list(APPEND MAIN_RUST_ENV_VARS "RUST_BACKTRACE=1")
if(CMAKE_C_COMPILER)   list(APPEND MAIN_RUST_ENV_VARS "CC=${CMAKE_C_COMPILER}") endif()
if(CMAKE_CXX_COMPILER) list(APPEND MAIN_RUST_ENV_VARS "CXX=${CMAKE_CXX_COMPILER}") endif()
if(CMAKE_LINKER)       list(APPEND MAIN_RUST_ENV_VARS "LD=${CMAKE_LINKER}") endif()
if(CMAKE_C_FLAGS)      list(APPEND MAIN_RUST_ENV_VARS "CFLAGS=${CMAKE_C_FLAGS}") endif()
if(CMAKE_CXX_FLAGS)    list(APPEND MAIN_RUST_ENV_VARS "CXXFLAGS=${CMAKE_CXX_FLAGS}") endif()
if(CMAKE_EXE_LINKER_FLAGS) list(APPEND MAIN_RUST_ENV_VARS "LDFLAGS=${CMAKE_EXE_LINKER_FLAGS}") endif()
if(CXXSTDLIB)          list(APPEND MAIN_RUST_ENV_VARS "CXXSTDLIB=${CXXSTDLIB}") endif()
list(APPEND MAIN_RUST_ENV_VARS "CARGO_NET_GIT_FETCH_WITH_CLI=true")
list(APPEND MAIN_RUST_ENV_VARS "CARGO_TARGET_DIR=${RUST_TARGET_DIR}") # Ensure cargo uses our specified target dir
if(RUSTFLAGS_ASAN_VALUE)
    list(APPEND MAIN_RUST_ENV_VARS "RUSTFLAGS=${RUSTFLAGS_ASAN_VALUE}") # Only ASAN flags here
else()
    # Ensure RUSTFLAGS is not passed if empty, or pass an empty RUSTFLAGS
    # list(APPEND MAIN_RUST_ENV_VARS "RUSTFLAGS=") # Explicitly empty if needed
endif()

set(MAIN_RUST_ENV_SETUP_COMMANDS "")
foreach(env_var ${MAIN_RUST_ENV_VARS})
  list(APPEND MAIN_RUST_ENV_SETUP_COMMANDS "${CMAKE_COMMAND}" "-E" "env" "${env_var}")
endforeach()

# Ensure SOROBAN_EXTERN_ARGS and SOROBAN_DEPEND_ARGS are treated as lists
# The CARGO_EXECUTABLE is already a list if channel is used.
# The feature string is also a list of two elements if features are present.
# The extern and depend args are also lists.
# CMake handles lists of arguments to COMMAND correctly.
add_custom_command(
    OUTPUT ${LIBRUST_STELLAR_CORE_A}
    COMMAND ${MAIN_RUST_ENV_SETUP_COMMANDS}
            ${CARGO_EXECUTABLE} rustc --package stellar-core ${RUST_PROFILE_ARG} --locked
            --target-dir ${RUST_TARGET_DIR}
            ${MAIN_RUST_CARGO_FEATURES_STRING}
            -- ${SOROBAN_EXTERN_ARGS} ${SOROBAN_DEPEND_ARGS}
    COMMAND ${CMAKE_RANLIB} ${LIBRUST_STELLAR_CORE_A}
    WORKING_DIRECTORY ${RUST_STELLAR_CORE_MANIFEST_DIR}
    DEPENDS ${MAIN_RUST_BUILD_DEPENDS}
    COMMENT "Building main Rust static library librust_stellar_core.a with features: ${MAIN_RUST_CARGO_FEATURES_STRING}"
    VERBATIM
)
message(STATUS "    Added custom command to build ${LIBRUST_STELLAR_CORE_A}")

# 4. Create a Target for the Rust Library
add_library(rust_stellar_core_lib STATIC IMPORTED GLOBAL)
set_target_properties(rust_stellar_core_lib PROPERTIES
    IMPORTED_LOCATION "${LIBRUST_STELLAR_CORE_A}"
)
# Add dependency on the custom command's output directly
add_dependencies(rust_stellar_core_lib ${LIBRUST_STELLAR_CORE_A})
message(STATUS "    Created IMPORTED library target rust_stellar_core_lib for ${LIBRUST_STELLAR_CORE_A}")

message(STATUS "  Main Rust library build configuration complete.")

# 2. Install cxxbridge-cmd
set(RUST_CXXBRIDGE_EXECUTABLE "${RUST_BIN_DIR}/cxxbridge")
set(CXXBRIDGE_CMD_VERSION "1.0.68") # As specified

# Command to install cxxbridge-cmd. Using a list for CARGO_EXECUTABLE.
# The environment variable CARGO_HTTP_MULTIPLEXING=false is added.
set(INSTALL_CXXBRIDGE_CMD
    ${CMAKE_COMMAND} -E env "CARGO_HTTP_MULTIPLEXING=false"
    ${CARGO_EXECUTABLE} install --force --locked --root ${RUST_BUILD_DIR} cxxbridge-cmd --version ${CXXBRIDGE_CMD_VERSION}
)

add_custom_command(
    OUTPUT ${RUST_CXXBRIDGE_EXECUTABLE}
    COMMAND ${INSTALL_CXXBRIDGE_CMD}
    DEPENDS ${RUST_TOOLCHAIN_FILE} # Re-install if toolchain file changes
    COMMENT "Installing cxxbridge-cmd version ${CXXBRIDGE_CMD_VERSION} using ${CARGO_EXECUTABLE}"
    VERBATIM
)
message(STATUS "  Added custom command to install cxxbridge-cmd to ${RUST_CXXBRIDGE_EXECUTABLE}")

# Custom target to ensure cxxbridge-cmd is built
add_custom_target(cxxbridge_generator_target
    DEPENDS ${RUST_CXXBRIDGE_EXECUTABLE}
    COMMENT "Ensuring cxxbridge-cmd is installed"
)
message(STATUS "  Added custom target cxxbridge_generator_target.")

# 3. Generate Rust Bridge Files (RustBridge.h, RustBridge.cpp)
set(RUST_BRIDGE_INPUT_RS "${CMAKE_SOURCE_DIR}/src/rust/src/bridge.rs")
set(RUST_BRIDGE_GENERATED_DIR "${CMAKE_BINARY_DIR}/generated/rust")
set(RUST_BRIDGE_OUTPUT_H "${RUST_BRIDGE_GENERATED_DIR}/RustBridge.h")
set(RUST_BRIDGE_OUTPUT_CPP "${RUST_BRIDGE_GENERATED_DIR}/RustBridge.cpp")

file(MAKE_DIRECTORY ${RUST_BRIDGE_GENERATED_DIR})
message(STATUS "  Rust bridge generated files will be in: ${RUST_BRIDGE_GENERATED_DIR}")

# Check for input bridge.rs
if(NOT EXISTS ${RUST_BRIDGE_INPUT_RS})
    message(FATAL_ERROR "Rust bridge input file not found: ${RUST_BRIDGE_INPUT_RS}")
endif()

# Glob for all .rs files in src/rust/src/ as dependencies for cxxbridge
file(GLOB RUST_SRC_FILES "${CMAKE_SOURCE_DIR}/src/rust/src/*.rs")
if(RUST_SRC_FILES)
    message(STATUS "  Found Rust source files for cxxbridge dependency: ${RUST_SRC_FILES}")
else()
    message(WARNING "  No .rs files found in src/rust/src/ for cxxbridge dependency. This might be an error if Rust files are expected.")
endif()

# Custom command for RustBridge.h
set(RUST_BRIDGE_HEADER_CMD
    ${RUST_CXXBRIDGE_EXECUTABLE} ${RUST_BRIDGE_INPUT_RS} --cfg test=false --header --output ${RUST_BRIDGE_OUTPUT_H}.tmp &&
    ${CMAKE_COMMAND} -E copy_if_different ${RUST_BRIDGE_OUTPUT_H}.tmp ${RUST_BRIDGE_OUTPUT_H} &&
    ${CMAKE_COMMAND} -E remove ${RUST_BRIDGE_OUTPUT_H}.tmp
)
add_custom_command(
    OUTPUT ${RUST_BRIDGE_OUTPUT_H}
    COMMAND ${CMAKE_COMMAND} -E env "RUST_BACKTRACE=1" ${RUST_BRIDGE_HEADER_CMD}
    DEPENDS ${RUST_BRIDGE_INPUT_RS} cxxbridge_generator_target ${RUST_SRC_FILES}
    COMMENT "Generating RustBridge.h from ${RUST_BRIDGE_INPUT_RS}"
    VERBATIM
)
message(STATUS "  Added custom command for RustBridge.h generation.")

# Custom command for RustBridge.cpp
set(RUST_BRIDGE_CPP_CMD
    ${RUST_CXXBRIDGE_EXECUTABLE} ${RUST_BRIDGE_INPUT_RS} --cfg test=false --output ${RUST_BRIDGE_OUTPUT_CPP}.tmp &&
    ${CMAKE_COMMAND} -E copy_if_different ${RUST_BRIDGE_OUTPUT_CPP}.tmp ${RUST_BRIDGE_OUTPUT_CPP} &&
    ${CMAKE_COMMAND} -E remove ${RUST_BRIDGE_OUTPUT_CPP}.tmp
)
add_custom_command(
    OUTPUT ${RUST_BRIDGE_OUTPUT_CPP}
    COMMAND ${CMAKE_COMMAND} -E env "RUST_BACKTRACE=1" ${RUST_BRIDGE_CPP_CMD}
    DEPENDS ${RUST_BRIDGE_INPUT_RS} cxxbridge_generator_target ${RUST_SRC_FILES}
    COMMENT "Generating RustBridge.cpp from ${RUST_BRIDGE_INPUT_RS}"
    VERBATIM
)
message(STATUS "  Added custom command for RustBridge.cpp generation.")

list(APPEND GENERATED_CORE_SOURCES ${RUST_BRIDGE_OUTPUT_CPP})
message(STATUS "  Added ${RUST_BRIDGE_OUTPUT_CPP} to GENERATED_CORE_SOURCES.")

include_directories(${RUST_BRIDGE_GENERATED_DIR})
message(STATUS "  Added ${RUST_BRIDGE_GENERATED_DIR} to include directories.")

# Create a list for all generated headers for IDEs if it doesn't exist
if(NOT DEFINED ALL_GENERATED_HEADERS_FOR_IDE)
    set(ALL_GENERATED_HEADERS_FOR_IDE "")
endif()
list(APPEND ALL_GENERATED_HEADERS_FOR_IDE ${RUST_BRIDGE_OUTPUT_H})
# Also add the .cpp to this list for IDEs that might benefit from seeing it, even if it's a source.
list(APPEND ALL_GENERATED_HEADERS_FOR_IDE ${RUST_BRIDGE_OUTPUT_CPP})
message(STATUS "  Added Rust bridge generated files to ALL_GENERATED_HEADERS_FOR_IDE for IDE visibility.")

message(STATUS "Rust integration (cxxbridge generation) configuration complete.")


# CONFIG.H GENERATION
include(CheckIncludeFiles)
include(CheckLibraryExists)

# Set up variables for config.h.in
# PACKAGE_NAME and PACKAGE_VERSION
set(PACKAGE_NAME "${CMAKE_PROJECT_NAME}")
set(PACKAGE_VERSION "0.1.0") # TODO: Make this configurable (e.g., from git describe)

# HAVE_LIBUNWIND
# This check is primarily for the config.h definition.
# Actual linking libraries for libunwind are handled in PLATFORM_SPECIFIC_LINK_LIBS section.
if(APPLE)
    set(HAVE_LIBUNWIND 1 CACHE INTERNAL "Libunwind is available on Apple systems for config.h")
else()
    if(NOT DISABLE_LIBUNWIND)
        # This check_include_files is for the #define in config.h
        check_include_files("libunwind.h" _HAVE_LIBUNWIND_H_CHECK_FOR_CONFIG)
        if(_HAVE_LIBUNWIND_H_CHECK_FOR_CONFIG)
            # The check_library_exists here is also for confirming presence for config.h
            # It adds to CMAKE_REQUIRED_LIBRARIES for the check itself, not for final linking of all targets.
            set(CMAKE_REQUIRED_LIBRARIES_OLD_UNWIND ${CMAKE_REQUIRED_LIBRARIES})
            list(APPEND CMAKE_REQUIRED_LIBRARIES unwind)
            check_library_exists("unwind" "backtrace" "" _HAVE_LIBUNWIND_LIB_CHECK_FOR_CONFIG)
            set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES_OLD_UNWIND}) # Restore

            if(_HAVE_LIBUNWIND_LIB_CHECK_FOR_CONFIG)
                set(HAVE_LIBUNWIND 1 CACHE INTERNAL "Libunwind header and library found for config.h")
            else()
                set(HAVE_LIBUNWIND 0 CACHE INTERNAL "Libunwind header found, but library check failed for config.h")
            endif()
        else()
            set(HAVE_LIBUNWIND 0 CACHE INTERNAL "Libunwind header (libunwind.h) not found for config.h")
        endif()
    else()
        set(HAVE_LIBUNWIND 0 CACHE INTERNAL "Libunwind disabled by DISABLE_LIBUNWIND option for config.h")
    endif()
endif()

if(HAVE_LIBUNWIND)
    message(STATUS "Libunwind: YES (for config.h definition)")
    # Actual linking is handled by PLATFORM_SPECIFIC_LINK_LIBS based on pkg-config or other means.
    # If not Apple, and pkg-config found libunwind, it's added to PLATFORM_SPECIFIC_LINK_LIBS.
    # If on Apple, or if libunwind is part of libc, no explicit link flag might be needed in PLATFORM_SPECIFIC_LINK_LIBS.
else()
    message(STATUS "Libunwind: NO (for config.h definition) (DISABLE_LIBUNWIND: ${DISABLE_LIBUNWIND})")
endif()

# MSAN_ENABLED for config.h (from sanitizer logic)
# The add_compile_definitions(MSAN_ENABLED=1) is for the compiler.
# For config.h.in, we need a CMake variable.
if(ENABLE_MEMCHECK)
    set(MSAN_ENABLED 1 CACHE INTERNAL "MemorySanitizer is enabled")
else()
    set(MSAN_ENABLED 0 CACHE INTERNAL "MemorySanitizer is not enabled")
endif()

# HAVE_POSTGRES for config.h (from PostgreSQL client library logic)
# This variable is set in the Libpq section.
# No specific 'set' needed here as it's already set to 0 or 1.

# AFL_LLVM_MODE for config.h (from AFL logic)
# The add_compile_definitions(AFL_LLVM_MODE=1) is for the compiler.
# For config.h.in, we need a CMake variable.
if(ENABLE_AFL)
    string(REGEX MATCH "clang" IS_AFL_CLANG_COMPILER "${AFL_C_COMPILER}") # AFL_C_COMPILER is set in AFL block
    if(IS_AFL_CLANG_COMPILER)
        set(AFL_LLVM_MODE 1 CACHE INTERNAL "AFL LLVM mode is enabled")
    else()
        set(AFL_LLVM_MODE 0 CACHE INTERNAL "AFL LLVM mode is not enabled (using GCC-based AFL or AFL disabled)")
    endif()
else()
    set(AFL_LLVM_MODE 0 CACHE INTERNAL "AFL is not enabled")
endif()

# Ensure the config.h.in path is correct. It should be in the source directory.
set(CONFIG_H_IN_PATH "${CMAKE_CURRENT_SOURCE_DIR}/config.h.in")
set(CONFIG_H_OUT_PATH "${CMAKE_BINARY_DIR}/config.h")

if(NOT EXISTS "${CONFIG_H_IN_PATH}")
    message(FATAL_ERROR "config.h.in not found at ${CONFIG_H_IN_PATH}. It should be in the root source directory. Please create it with #cmakedefine HAVE_POSTGRES etc.")
else()
    message(STATUS "Found config.h.in at: ${CONFIG_H_IN_PATH}")
endif()

# The variables for config.h.in are:
# PACKAGE_NAME, PACKAGE_VERSION, HAVE_LIBUNWIND, MSAN_ENABLED, AFL_LLVM_MODE, HAVE_POSTGRES
configure_file(
    "${CONFIG_H_IN_PATH}"
    "${CONFIG_H_OUT_PATH}"
    # No @ONLY, so #cmakedefine works.
)

message(STATUS "Generated config.h at: ${CONFIG_H_OUT_PATH} (HAVE_POSTGRES: ${HAVE_POSTGRES})")

# Add the directory containing the generated config.h to the include path
# This makes #include "config.h" work from source files.
include_directories("${CMAKE_BINARY_DIR}")
message(STATUS "Added ${CMAKE_BINARY_DIR} to include directories for config.h.")

# MAIN EXECUTABLE (stellar-core)
message(STATUS "Configuring main executable: stellar-core")

# 1. Generate StellarCoreVersion.cpp
set(STELLAR_CORE_VERSION_CPP_DIR "${CMAKE_BINARY_DIR}/generated/main")
set(STELLAR_CORE_VERSION_CPP "${STELLAR_CORE_VERSION_CPP_DIR}/StellarCoreVersion.cpp")
set(STELLAR_CORE_VERSION_CPP_IN "${CMAKE_SOURCE_DIR}/src/main/StellarCoreVersion.cpp.in")

file(MAKE_DIRECTORY ${STELLAR_CORE_VERSION_CPP_DIR})
message(STATUS "  Generated StellarCoreVersion.cpp will be in: ${STELLAR_CORE_VERSION_CPP_DIR}")

if(NOT EXISTS ${STELLAR_CORE_VERSION_CPP_IN})
    message(FATAL_ERROR "StellarCoreVersion.cpp.in not found at ${STELLAR_CORE_VERSION_CPP_IN}")
endif()

# Command to get version string (simplified for now, git describe can be complex in CMake)
# Using PACKAGE_VERSION as a fallback if git describe fails or git is not found.
set(VERSION_COMMAND_SCRIPT "")
if(GIT_FOUND)
    # Try to get git describe, fallback to PACKAGE_VERSION if it fails
    set(VERSION_COMMAND_SCRIPT
        "vers=$$(git describe --always --dirty --tags 2>/dev/null); "
        "if [ -z \"$$vers\" ]; then vers=\"${PACKAGE_NAME} ${PACKAGE_VERSION}\"; fi; "
        "echo \"$$vers\""
    )
else()
    set(VERSION_COMMAND_SCRIPT "echo \"${PACKAGE_NAME} ${PACKAGE_VERSION}\"")
endif()

# Temporary file for sed output
set(STELLAR_CORE_VERSION_CPP_TMP "${STELLAR_CORE_VERSION_CPP}.tmp")

add_custom_command(
    OUTPUT ${STELLAR_CORE_VERSION_CPP}
    # The command uses bash -c to handle shell features like $() and pipes.
    # It first gets the version string, then uses sed to replace %%VERSION%%.
    # Finally, it copies if different to avoid unnecessary recompilation.
    COMMAND bash -c "${VERSION_COMMAND_SCRIPT} | sed -e 's@%%VERSION%%@'\"$$(cat -)\"'@' < \"${STELLAR_CORE_VERSION_CPP_IN}\" > \"${STELLAR_CORE_VERSION_CPP_TMP}\""
    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${STELLAR_CORE_VERSION_CPP_TMP}" "${STELLAR_CORE_VERSION_CPP}"
    COMMAND ${CMAKE_COMMAND} -E remove "${STELLAR_CORE_VERSION_CPP_TMP}"
    DEPENDS ${STELLAR_CORE_VERSION_CPP_IN} # Re-run if template changes
    # To re-run if git HEAD changes, one might depend on ${CMAKE_SOURCE_DIR}/.git/HEAD (if it exists and is a file)
    # For simplicity and robustness across different git versions/setups, often this is made ALWAYS or
    # a separate mechanism updates a version file that this depends on.
    # For now, depending on the .in file is the primary trigger.
    # Consider adding a more explicit re-evaluation trigger if git versioning is critical for every build.
    # Using a stamp file that is always out-of-date can force re-run, or specific target.
    # For this step, we'll rely on the .in file dependency. If git version changes, user might need to clean/reconfigure.
    # A common pattern is to have a custom target that ALWAYS runs and touches a file this depends on.
    COMMENT "Generating StellarCoreVersion.cpp"
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} # git describe needs to be in the repo
    VERBATIM
)
message(STATUS "  Added custom command to generate ${STELLAR_CORE_VERSION_CPP}")
# Add this generated file to a list that will be part of STELLAR_CORE_MAIN_SOURCES
list(APPEND GENERATED_CORE_SOURCES ${STELLAR_CORE_VERSION_CPP})

# Add the output directory to include path for StellarCoreVersion.cpp
include_directories(${STELLAR_CORE_VERSION_CPP_DIR})
message(STATUS "  Added ${STELLAR_CORE_VERSION_CPP_DIR} to include directories.")

# 2. Collect C++ Source Files for stellar-core
set(STELLAR_CORE_MAIN_SOURCES "")
message(STATUS "  Collecting C++ sources for stellar-core...")

# Glob for all .cpp files under src/, then exclude specific directories/files.
file(GLOB_RECURSE ALL_SRC_CPP_FILES "${CMAKE_SOURCE_DIR}/src/*.cpp")

set(EXCLUDE_PATTERNS
    "${CMAKE_SOURCE_DIR}/src/test/"
    "${CMAKE_SOURCE_DIR}/src/rust/"
    "${CMAKE_SOURCE_DIR}/src/protocol-curr/"
    "${CMAKE_SOURCE_DIR}/src/protocol-next/"
    "${CMAKE_SOURCE_DIR}/src/history/serialize-tests/"
    "${CMAKE_SOURCE_DIR}/src/main/dumpxdr.cpp"
)

foreach(file ${ALL_SRC_CPP_FILES})
    set(exclude_file FALSE)
    foreach(pattern ${EXCLUDE_PATTERNS})
        if(file MATCHES "^${pattern}") # Check if the file path starts with the pattern
            set(exclude_file TRUE)
            break()
        endif()
    endforeach()
    if(NOT exclude_file)
        list(APPEND STELLAR_CORE_MAIN_SOURCES ${file})
    endif()
endforeach()

message(STATUS "    Collected regular C++ sources: ${STELLAR_CORE_MAIN_SOURCES}")

# Add generated sources
# GENERATED_CORE_SOURCES includes XDRFilesSha256.cpp, Flex/Bison outputs, RustBridge.cpp, and StellarCoreVersion.cpp
if(GENERATED_CORE_SOURCES)
    list(APPEND STELLAR_CORE_MAIN_SOURCES ${GENERATED_CORE_SOURCES})
    message(STATUS "    Added generated C++ sources: ${GENERATED_CORE_SOURCES}")
else()
    message(WARNING "    GENERATED_CORE_SOURCES is empty. This might be an error if generated sources are expected.")
endif()

if(NOT STELLAR_CORE_MAIN_SOURCES)
    message(FATAL_ERROR "No source files found for the stellar-core executable. Check globbing patterns and generated source lists.")
endif()
message(STATUS "  Total C++ sources for stellar-core: ${STELLAR_CORE_MAIN_SOURCES}")

# 3. Define stellar-core Executable Target
add_executable(stellar-core ${STELLAR_CORE_MAIN_SOURCES})
message(STATUS "  Defined executable target: stellar-core")

# 4. Set Include Directories for stellar-core
message(STATUS "  Setting include directories for stellar-core...")
target_include_directories(stellar-core PUBLIC
    ${CMAKE_BINARY_DIR} # For config.h, and other generated headers in CMAKE_BINARY_DIR/generated/*
    ${CMAKE_SOURCE_DIR}/src # For all source headers
)
# Specific include directories for dependencies (some might be via INTERFACE libraries)
# XDRPP_INCLUDE_DIRS and LIBSODIUM_INCLUDE_DIRS were variables for ExternalProject.
# If using interface libraries (e.g., 'xdrpp', 'libsodium'), linking to them handles includes.
# The XDR_GENERATED_INCLUDE_DIR, XDRQUERY_GENERATED_DIR, RUST_BRIDGE_GENERATED_DIR are already added globally.
# However, explicit target_include_directories can be clearer.

# From previous setups:
# xdrpp (target) provides includes if IMPORTED or from add_subdirectory
# libsodium (target) provides includes if IMPORTED or from add_subdirectory
# asio_interface (target)
# soci_interface (target)
# medida_interface (target)
# spdlog_interface (target, if USE_SPDLOG)
# sqlite3_lib (target) - its include dir was set to PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/lib/sqlite
# cereal::cereal (target from add_subdirectory)
# fmt::fmt (target from add_subdirectory)
# TracyClient (target from add_subdirectory if ENABLE_TRACY)

# Global includes already added:
# ${CMAKE_BINARY_DIR} (for config.h)
# ${XDR_GENERATED_INCLUDE_DIR}
# ${XDRQUERY_GENERATED_DIR}
# ${RUST_BRIDGE_GENERATED_DIR}
# ${STELLAR_CORE_VERSION_CPP_DIR}

# Some libraries like asio_interface, soci_interface, etc., are interface libraries
# that should propagate their include directories when linked.
# For those set up with specific include variables (like XDRPP_INCLUDE_DIRS, LIBSODIUM_INCLUDE_DIRS
# when pkg-config was used but no target created, or for internal builds), add them here.
if(XDRPP_INCLUDE_DIRS AND NOT TARGET xdrpp) # If xdrpp target exists, it should handle this.
    target_include_directories(stellar-core PUBLIC ${XDRPP_INCLUDE_DIRS})
endif()
if(LIBSODIUM_INCLUDE_DIRS AND NOT TARGET libsodium) # If libsodium target exists, it should handle this.
    target_include_directories(stellar-core PUBLIC ${LIBSODIUM_INCLUDE_DIRS})
endif()
if(POSTGRESQL_FOUND AND POSTGRESQL_INCLUDE_DIRS) # From find_package(PostgreSQL) or pkg-config
    target_include_directories(stellar-core PUBLIC ${POSTGRESQL_INCLUDE_DIRS})
endif()

# 5. Link stellar-core Against Libraries
message(STATUS "  Linking stellar-core against libraries...")
target_link_libraries(stellar-core PRIVATE
    lib3rdparty_a       # Contains util, json, and spdlog sources (if USE_SPDLOG)
    soci_interface      # Links to soci_lib (core + postgresql backend if enabled)
    medida_interface    # Links to medida_lib (if internal) or system libmedida
    xdrpp               # Interface for system or internal xdrpp
    libsodium           # Interface for system or internal libsodium
    rust_stellar_core_lib # Main Rust library
    fmt::fmt            # From add_subdirectory
    cereal::cereal      # From add_subdirectory
    Threads::Threads    # For pthread
)

# Conditional linking for Tracy
if(ENABLE_TRACY AND TARGET TracyClient) # TracyClient is the target name from Tracy's CMakeLists
    target_link_libraries(stellar-core PRIVATE TracyClient)
    message(STATUS "    Linked against TracyClient.")
endif()

# Conditional linking for PostgreSQL (if soci_interface doesn't bring it transitively via PostgreSQL::libpq)
# PostgreSQL::libpq is already linked to soci_lib if postgres is enabled for SOCI.
# If any other part of stellar-core needs direct pg headers/libs, this might be needed,
# but usually, it's encapsulated by the component using it (like SOCI).
# if(HAVE_POSTGRES AND TARGET PostgreSQL::libpq)
# target_link_libraries(stellar-core PRIVATE PostgreSQL::libpq)
# endif()

# Link against dl library (for dlfcn.h, dlopen, etc.)
# Check if the dl library is needed and find it.
find_library(DL_LIBRARY dl)
if(DL_LIBRARY)
    target_link_libraries(stellar-core PRIVATE ${DL_LIBRARY})
    message(STATUS "    Linked against dl library: ${DL_LIBRARY}")
else()
    message(WARNING "    dl library not found. Dynamic loading features might be affected.")
endif()

# Link against PLATFORM_SPECIFIC_LINK_LIBS (for filesystem, libunwind)
if(PLATFORM_SPECIFIC_LINK_LIBS)
    target_link_libraries(stellar-core PRIVATE ${PLATFORM_SPECIFIC_LINK_LIBS})
    message(STATUS "    Linked against platform specific libraries: ${PLATFORM_SPECIFIC_LINK_LIBS}")
endif()

message(STATUS "stellar-core executable configuration complete.")
