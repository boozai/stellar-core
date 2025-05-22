cmake_minimum_required(VERSION 3.16)
project(libsodium_project LANGUAGES C)

message(STATUS "Processing lib/libsodium/CMakeLists.txt...")

include(CheckTypeSize)
include(CheckCSourceCompiles)

# --- Configure sodium_config.h ---
# Check for endianness (simplified)
check_c_source_compiles("
    const int i = 1;
    const char *p = (const char *)&i;
    int main(void) { return p[0] == 1 ? 0 : 1; }
" NATIVE_LITTLE_ENDIAN_CHECK)
if(NATIVE_LITTLE_ENDIAN_CHECK)
    set(NATIVE_LITTLE_ENDIAN 1 CACHE INTERNAL "System is little endian")
else()
    set(NATIVE_LITTLE_ENDIAN 0 CACHE INTERNAL "System is big endian (or unknown)")
endif()
message(STATUS "  NATIVE_LITTLE_ENDIAN: ${NATIVE_LITTLE_ENDIAN}")

# Check for weak symbols (simplified)
check_c_source_compiles("
    __attribute__((weak)) void foo(void) {}
    int main(void) { foo(); return 0; }
" HAVE_WEAK_SYMBOLS_CHECK)
if(HAVE_WEAK_SYMBOLS_CHECK)
    set(HAVE_WEAK_SYMBOLS 1 CACHE INTERNAL "System has weak symbols")
else()
    set(HAVE_WEAK_SYMBOLS 0 CACHE INTERNAL "System does not have weak symbols (or check failed)")
endif()
message(STATUS "  HAVE_WEAK_SYMBOLS: ${HAVE_WEAK_SYMBOLS}")

# Check for inttypes.h, stdint.h, stdbool.h
check_include_files("inttypes.h" HAVE_INTTYPES_H)
check_include_files("stdint.h" HAVE_STDINT_H)
check_include_files("stdbool.h" HAVE_STDBOOL_H)
message(STATUS "  HAVE_INTTYPES_H: ${HAVE_INTTYPES_H}")
message(STATUS "  HAVE_STDINT_H: ${HAVE_STDINT_H}")
message(STATUS "  HAVE_STDBOOL_H: ${HAVE_STDBOOL_H}")

# Get SIZEOF_SIZE_T
check_type_size("size_t" SIZEOF_SIZE_T)
if(NOT SIZEOF_SIZE_T)
    message(WARNING "  Could not determine SIZEOF_SIZE_T. Defaulting to 8 for sodium_config.h.in.")
    set(SIZEOF_SIZE_T 8) # Default if check fails
endif()
message(STATUS "  SIZEOF_SIZE_T: ${SIZEOF_SIZE_T}")

# Configure sodium_config.h from sodium_config.h.in (created in previous step)
# The .in file should be in CMAKE_CURRENT_SOURCE_DIR (lib/libsodium/)
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/sodium_config.h.in"
    "${CMAKE_CURRENT_BINARY_DIR}/sodium_config.h"
    @ONLY # Use @ONLY because .in file uses #cmakedefine
)
message(STATUS "  Configured sodium_config.h in ${CMAKE_CURRENT_BINARY_DIR}")


# --- Define sodium_lib (Static Library) ---
add_library(sodium_lib STATIC)
message(STATUS "  Defined static library target: sodium_lib")

set(SODIUM_ALL_SOURCES "")

# Glob sources (simplified approach - this will grab many files, including platform specific ones)
# A more precise approach would list sources explicitly or use more refined globs based on detected platform/features.
file(GLOB_RECURSE SODIUM_SOURCES_CORE RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "src/libsodium/sodium/*.c")
if(SODIUM_SOURCES_CORE) list(APPEND SODIUM_ALL_SOURCES ${SODIUM_SOURCES_CORE}) endif()

set(CRYPTO_DIRS
    crypto_aead crypto_auth crypto_box crypto_core crypto_generichash crypto_hash
    crypto_kdf crypto_kx crypto_onetimeauth crypto_pwhash crypto_scalarmult
    crypto_secretbox crypto_secretstream crypto_shorthash crypto_sign crypto_stream
    crypto_verify randombytes scalarmult sign stream
)
foreach(crypto_dir ${CRYPTO_DIRS})
    file(GLOB_RECURSE TEMP_CRYPTO_SOURCES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "src/libsodium/${crypto_dir}/*.c" "src/libsodium/${crypto_dir}/*/*.c")
    if(TEMP_CRYPTO_SOURCES) list(APPEND SODIUM_ALL_SOURCES ${TEMP_CRYPTO_SOURCES}) endif()
endforeach()

file(GLOB_RECURSE SODIUM_SOURCES_MINIMAL RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "src/libsodium/minimal/*.c")
if(SODIUM_SOURCES_MINIMAL) list(APPEND SODIUM_ALL_SOURCES ${SODIUM_SOURCES_MINIMAL}) endif()

list(REMOVE_DUPLICATES SODIUM_ALL_SOURCES)
message(STATUS "  Globbed ${SODIUM_ALL_SOURCES_COUNT} unique C source files for sodium_lib.") # Requires string(LENGTH ...) for actual count

target_sources(sodium_lib PRIVATE ${SODIUM_ALL_SOURCES})

target_include_directories(sodium_lib
    PUBLIC
        # For installed targets, this will point to <prefix>/include
        # For build tree usage, this points to src/libsodium/include
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/libsodium/include>
        $<INSTALL_INTERFACE:include> 
    PRIVATE
        ${CMAKE_CURRENT_BINARY_DIR} # For sodium_config.h
        ${CMAKE_CURRENT_SOURCE_DIR}/src/libsodium/include/sodium # For internal headers if any use #include "sodium/..."
        # Add other internal include paths if necessary, e.g., for different modules
        "${CMAKE_CURRENT_SOURCE_DIR}/src/libsodium/crypto_generichash/blake2b" # Example if some sources need this path
        "${CMAKE_CURRENT_SOURCE_DIR}/src/libsodium/crypto_sign/ed25519"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/libsodium/crypto_scalarmult/curve25519"
        # ... etc. This needs careful checking of libsodium's internal include structure.
        # The provided globbing is very broad, a real build would be more selective.
)

target_compile_definitions(sodium_lib PRIVATE
    SODIUM_STATIC       # Building as a static library
    SODIUM_EXPORT=      # Disable DLL export/import macros for static build
    DLL_EXPORT          # Another define sometimes used, ensure it's empty for static
    PIC                 # Should be handled by CMAKE_POSITION_INDEPENDENT_CODE
)
if(NATIVE_LITTLE_ENDIAN)
    target_compile_definitions(sodium_lib PRIVATE NATIVE_LITTLE_ENDIAN)
endif()
if(HAVE_WEAK_SYMBOLS)
    target_compile_definitions(sodium_lib PRIVATE HAVE_WEAK_SYMBOLS)
endif()
if(HAVE_INTTYPES_H)
    target_compile_definitions(sodium_lib PRIVATE HAVE_INTTYPES_H)
endif()
if(HAVE_STDINT_H)
    target_compile_definitions(sodium_lib PRIVATE HAVE_STDINT_H)
endif()
if(HAVE_STDBOOL_H)
    target_compile_definitions(sodium_lib PRIVATE HAVE_STDBOOL_H)
endif()
# Add other defines based on checks or simplifications (e.g., for specific CPU features if known)

message(STATUS "  Set include directories and compile definitions for sodium_lib.")

# --- Installation Rules ---
if(CMAKE_INSTALL_LIBDIR AND CMAKE_INSTALL_INCLUDEDIR)
    install(TARGETS sodium_lib
        EXPORT libsodium_targets # Export for find_package
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} # For DLLs on Windows, not applicable for static
    )
    
    # Install the public headers (the 'sodium' directory and 'sodium.h')
    install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/src/libsodium/include/sodium"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
        FILES_MATCHING PATTERN "*.h"
    )
    install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/libsodium/include/sodium.h"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    )
    # Install the generated sodium_config.h
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/sodium_config.h"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/sodium" # Place it inside the sodium dir like original
    )
    message(STATUS "    Added install rules for sodium_lib and headers.")

    install(EXPORT libsodium_targets
      FILE libsodium-targets.cmake
      NAMESPACE sodium:: # Use a namespace for imported targets
      DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libsodium
    )
    message(STATUS "    Added install(EXPORT ...) for libsodium_targets.")
else()
    message(WARNING "    CMAKE_INSTALL_LIBDIR or CMAKE_INSTALL_INCLUDEDIR not defined. Skipping install rules for sodium_lib.")
endif()

message(STATUS "Finished processing lib/libsodium/CMakeLists.txt.")
