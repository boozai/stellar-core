cmake_minimum_required(VERSION 3.16)

message(STATUS "Processing src/CMakeLists.txt...")

# Initialize lists for generated files specific to this subdirectory
set(GENERATED_CORE_SOURCES_SRC "") # Sources generated in this file for stellar-core, _SRC suffix to avoid any potential clash
set(XDR_GENERATED_HEADERS_SRC "")  # XDR headers specifically, _SRC suffix

# --- XDR CODE GENERATION ---
message(STATUS "  Configuring XDR code generation (from src/CMakeLists.txt)...")
set(XDR_FILES_IN_SRC "") # Local variable for XDR source files within src/
file(GLOB XDR_PROTOCOL_CURR_FILES_IN_SRC_CTX "${CMAKE_CURRENT_SOURCE_DIR}/protocol-curr/xdr/*.x")
if(XDR_PROTOCOL_CURR_FILES_IN_SRC_CTX) list(APPEND XDR_FILES_IN_SRC ${XDR_PROTOCOL_CURR_FILES_IN_SRC_CTX}) endif()
file(GLOB XDR_PROTOCOL_NEXT_FILES_IN_SRC_CTX "${CMAKE_CURRENT_SOURCE_DIR}/protocol-next/xdr/*.x")
if(XDR_PROTOCOL_NEXT_FILES_IN_SRC_CTX) list(APPEND XDR_FILES_IN_SRC ${XDR_PROTOCOL_NEXT_FILES_IN_SRC_CTX}) endif()

if(NOT XDR_FILES_IN_SRC)
    message(WARNING "    No .x files found in src/protocol-curr/xdr/ or src/protocol-next/xdr/ (from src/CMakeLists.txt).")
endif()

if(NOT XDR_GENERATED_INCLUDE_DIR) # Should be defined in root
    message(FATAL_ERROR "    XDR_GENERATED_INCLUDE_DIR is not defined in parent scope!")
endif()
if(NOT XDRC_EXECUTABLE) # Should be defined in root
    message(FATAL_ERROR "    XDRC_EXECUTABLE is not set in parent scope!")
endif()

foreach(XDR_FILE_FULL_PATH_CTX ${XDR_FILES_IN_SRC})
    get_filename_component(XDR_FILE_NAME_CTX ${XDR_FILE_FULL_PATH_CTX} NAME_WE)
    get_filename_component(XDR_FILE_NAME_WITH_EXT_CTX ${XDR_FILE_FULL_PATH_CTX} NAME)
    set(XDR_OUTPUT_HEADER_CTX "${XDR_GENERATED_INCLUDE_DIR}/${XDR_FILE_NAME_CTX}.h") # Use global XDR_GENERATED_INCLUDE_DIR

    set(XDRC_DEPENDENCY_CTX "")
    # XDRC_EXECUTABLE is set to "xdrc_executable" (a target name) in root if built internally.
    # Otherwise, it's a path to a system executable.
    # FORCE_INTERNAL_XDRPP is a good proxy for this.
    if(FORCE_INTERNAL_XDRPP)
        if(TARGET xdrc_executable) # Check if the target from lib/xdrpp/CMakeLists.txt exists
            set(XDRC_DEPENDENCY_CTX xdrc_executable)
            message(STATUS "      XDRC dependency for ${XDR_FILE_NAME_WITH_EXT_CTX} set to target xdrc_executable.")
        else()
            message(WARNING "      FORCE_INTERNAL_XDRPP is ON but target 'xdrc_executable' not found. XDR generation might not have correct dependency for xdrc tool.")
        endif()
    endif()
    # If XDRC_DEPENDENCY_CTX remains empty, the command depends only on the .x file and implicitly on XDRC_EXECUTABLE path being valid.
    
    add_custom_command(
        OUTPUT ${XDR_OUTPUT_HEADER_CTX}
        COMMAND ${XDRC_EXECUTABLE} -hh -pedantic -o ${XDR_OUTPUT_HEADER_CTX} ${XDR_FILE_FULL_PATH_CTX}
        DEPENDS ${XDR_FILE_FULL_PATH_CTX} ${XDRC_DEPENDENCY_CTX} # XDRC_DEPENDENCY_CTX will be xdrc_executable or empty
        COMMENT "Generating C++ header from ${XDR_FILE_NAME_WITH_EXT_CTX} (src/CMakeLists.txt)" VERBATIM)
    list(APPEND XDR_GENERATED_HEADERS_SRC ${XDR_OUTPUT_HEADER_CTX})
endforeach()

# XDRFilesSha256.cpp Generation
set(XDR_SHA256_CPP_FILE_OUT "${CMAKE_BINARY_DIR}/generated/XDRFilesSha256.cpp") # Consistent output path
set(HASH_XDRS_PROTOCOL_ARG_CTX "")
if(ENABLE_NEXT_PROTOCOL_VERSION_UNSAFE_FOR_PRODUCTION) set(HASH_XDRS_PROTOCOL_ARG_CTX "protocol-next") else() set(HASH_XDRS_PROTOCOL_ARG_CTX "protocol-curr") endif()
set(HASH_XDRS_SCRIPT_CTX "${CMAKE_SOURCE_DIR}/hash-xdrs.sh")

if(XDR_FILES_IN_SRC)
    add_custom_command(
        OUTPUT ${XDR_SHA256_CPP_FILE_OUT}
        COMMAND bash ${HASH_XDRS_SCRIPT_CTX} ${HASH_XDRS_PROTOCOL_ARG_CTX} > ${XDR_SHA256_CPP_FILE_OUT}
        DEPENDS ${XDR_FILES_IN_SRC} ${HASH_XDRS_SCRIPT_CTX}
        COMMENT "Generating ${XDR_SHA256_CPP_FILE_OUT} for ${HASH_XDRS_PROTOCOL_ARG_CTX} (src/CMakeLists.txt)" VERBATIM
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}) # Script expects to run from root project source
    list(APPEND GENERATED_CORE_SOURCES_SRC ${XDR_SHA256_CPP_FILE_OUT})
else()
    file(WRITE ${XDR_SHA256_CPP_FILE_OUT} "// No XDR sources found in src/, this file is intentionally empty (src/CMakeLists.txt).\nvoid xdrFilesSha256DummySrcCMake() {}\n")
    list(APPEND GENERATED_CORE_SOURCES_SRC ${XDR_SHA256_CPP_FILE_OUT})
endif()
message(STATUS "  XDR code generation configured (from src/CMakeLists.txt).")

# --- FLEX/BISON XDR QUERY GENERATION ---
message(STATUS "  Configuring Flex/Bison for XDR Query parsing (from src/CMakeLists.txt)...")
set(XDRQUERY_LEXER_INPUT_CTX "${CMAKE_CURRENT_SOURCE_DIR}/util/xdrquery/XDRQueryScanner.ll")
# XDRQUERY_GENERATED_DIR is defined and added to include_directories in root CMakeLists.txt
set(XDRQUERY_LEXER_OUTPUT_CPP_CTX "${XDRQUERY_GENERATED_DIR}/XDRQueryScanner.cpp")
set(XDRQUERY_PARSER_INPUT_CTX "${CMAKE_CURRENT_SOURCE_DIR}/util/xdrquery/XDRQueryParser.yy")
set(XDRQUERY_PARSER_OUTPUT_CPP_CTX "${XDRQUERY_GENERATED_DIR}/XDRQueryParser.cpp")
set(XDRQUERY_PARSER_OUTPUT_H_CTX "${XDRQUERY_GENERATED_DIR}/XDRQueryParser.h")

add_custom_command(
    OUTPUT ${XDRQUERY_LEXER_OUTPUT_CPP_CTX}
    COMMAND ${FLEX_EXECUTABLE} --outfile=${XDRQUERY_LEXER_OUTPUT_CPP_CTX} ${XDRQUERY_LEXER_INPUT_CTX}
    DEPENDS ${XDRQUERY_LEXER_INPUT_CTX} COMMENT "Generating scanner for XDRQuery (src/CMakeLists.txt)" VERBATIM)
add_custom_command(
    OUTPUT ${XDRQUERY_PARSER_OUTPUT_CPP_CTX} ${XDRQUERY_PARSER_OUTPUT_H_CTX}
    COMMAND ${BISON_EXECUTABLE} --defines=${XDRQUERY_PARSER_OUTPUT_H_CTX} --output=${XDRQUERY_PARSER_OUTPUT_CPP_CTX} ${XDRQUERY_PARSER_INPUT_CTX}
    DEPENDS ${XDRQUERY_PARSER_INPUT_CTX} COMMENT "Generating parser for XDRQuery (src/CMakeLists.txt)" VERBATIM)
list(APPEND GENERATED_CORE_SOURCES_SRC ${XDRQUERY_LEXER_OUTPUT_CPP_CTX} ${XDRQUERY_PARSER_OUTPUT_CPP_CTX})
message(STATUS "  Flex/Bison XDR Query generation configured (from src/CMakeLists.txt).")

# --- StellarCoreVersion.cpp Generation ---
message(STATUS "  Configuring StellarCoreVersion.cpp generation (from src/CMakeLists.txt)...")
# STELLAR_CORE_VERSION_CPP_DIR defined and added to include_directories in root
set(STELLAR_CORE_VERSION_CPP_IN_CTX "${CMAKE_CURRENT_SOURCE_DIR}/main/StellarCoreVersion.cpp.in")
set(STELLAR_CORE_VERSION_CPP_OUT_CTX "${STELLAR_CORE_VERSION_CPP_DIR}/StellarCoreVersion.cpp")

set(VERSION_COMMAND_SCRIPT_CTX "")
if(GIT_FOUND) set(VERSION_COMMAND_SCRIPT_CTX "vers=$$(git describe --always --dirty --tags 2>/dev/null); if [ -z \"$$vers\" ]; then vers=\"${PACKAGE_NAME} ${PACKAGE_VERSION}\"; fi; echo \"$$vers\"")
else() set(VERSION_COMMAND_SCRIPT_CTX "echo \"${PACKAGE_NAME} ${PACKAGE_VERSION}\"") endif()
set(STELLAR_CORE_VERSION_CPP_TMP_CTX "${STELLAR_CORE_VERSION_CPP_OUT_CTX}.tmp")

add_custom_command(
    OUTPUT ${STELLAR_CORE_VERSION_CPP_OUT_CTX}
    COMMAND bash -c "${VERSION_COMMAND_SCRIPT_CTX} | sed -e 's@%%VERSION%%@'\"$$(cat -)\"'@' < \"${STELLAR_CORE_VERSION_CPP_IN_CTX}\" > \"${STELLAR_CORE_VERSION_CPP_TMP_CTX}\""
    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${STELLAR_CORE_VERSION_CPP_TMP_CTX}" "${STELLAR_CORE_VERSION_CPP_OUT_CTX}"
    COMMAND ${CMAKE_COMMAND} -E remove "${STELLAR_CORE_VERSION_CPP_TMP_CTX}"
    DEPENDS ${STELLAR_CORE_VERSION_CPP_IN_CTX}
    COMMENT "Generating StellarCoreVersion.cpp (src/CMakeLists.txt)" WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} VERBATIM)
list(APPEND GENERATED_CORE_SOURCES_SRC ${STELLAR_CORE_VERSION_CPP_OUT_CTX})
message(STATUS "  StellarCoreVersion.cpp generation configured (from src/CMakeLists.txt).")

# --- Rust Bridge Files Generation ---
message(STATUS "  Configuring Rust bridge files generation (from src/CMakeLists.txt)...")
# RUST_CXXBRIDGE_EXECUTABLE, cxxbridge_generator_target, RUST_SRC_FILES (all .rs in src/rust/src) are from root
# RUST_BRIDGE_GENERATED_DIR is defined and added to include_directories in root
set(RUST_BRIDGE_INPUT_RS_CTX "${CMAKE_CURRENT_SOURCE_DIR}/rust/src/bridge.rs")
set(RUST_BRIDGE_OUTPUT_H_CTX "${RUST_BRIDGE_GENERATED_DIR}/RustBridge.h")
set(RUST_BRIDGE_OUTPUT_CPP_CTX "${RUST_BRIDGE_GENERATED_DIR}/RustBridge.cpp")

set(RUST_BRIDGE_HEADER_CMD_CTX "${RUST_CXXBRIDGE_EXECUTABLE}" "${RUST_BRIDGE_INPUT_RS_CTX}" --cfg test=false --header --output "${RUST_BRIDGE_OUTPUT_H_CTX}.tmp" && "${CMAKE_COMMAND}" -E copy_if_different "${RUST_BRIDGE_OUTPUT_H_CTX}.tmp" "${RUST_BRIDGE_OUTPUT_H_CTX}" && "${CMAKE_COMMAND}" -E remove "${RUST_BRIDGE_OUTPUT_H_CTX}.tmp")
add_custom_command(OUTPUT ${RUST_BRIDGE_OUTPUT_H_CTX} COMMAND ${CMAKE_COMMAND} -E env "RUST_BACKTRACE=1" ${RUST_BRIDGE_HEADER_CMD_CTX}
    DEPENDS ${RUST_BRIDGE_INPUT_RS_CTX} cxxbridge_generator_target ${RUST_SRC_FILES}
    COMMENT "Generating RustBridge.h (src/CMakeLists.txt)" VERBATIM)

set(RUST_BRIDGE_CPP_CMD_CTX "${RUST_CXXBRIDGE_EXECUTABLE}" "${RUST_BRIDGE_INPUT_RS_CTX}" --cfg test=false --output "${RUST_BRIDGE_OUTPUT_CPP_CTX}.tmp" && "${CMAKE_COMMAND}" -E copy_if_different "${RUST_BRIDGE_OUTPUT_CPP_CTX}.tmp" "${RUST_BRIDGE_OUTPUT_CPP_CTX}" && "${CMAKE_COMMAND}" -E remove "${RUST_BRIDGE_OUTPUT_CPP_CTX}.tmp")
add_custom_command(OUTPUT ${RUST_BRIDGE_OUTPUT_CPP_CTX} COMMAND ${CMAKE_COMMAND} -E env "RUST_BACKTRACE=1" ${RUST_BRIDGE_CPP_CMD_CTX}
    DEPENDS ${RUST_BRIDGE_INPUT_RS_CTX} cxxbridge_generator_target ${RUST_SRC_FILES}
    COMMENT "Generating RustBridge.cpp (src/CMakeLists.txt)" VERBATIM)
list(APPEND GENERATED_CORE_SOURCES_SRC ${RUST_BRIDGE_OUTPUT_CPP_CTX})
# Add to ALL_GENERATED_HEADERS_FOR_IDE in root scope if needed.
# For now, this header is primarily consumed by stellar-core target defined below.
if(DEFINED ALL_GENERATED_HEADERS_FOR_IDE)
    set_property(GLOBAL APPEND PROPERTY ALL_GENERATED_HEADERS_FOR_IDE ${RUST_BRIDGE_OUTPUT_H_CTX})
endif()
message(STATUS "  Rust bridge file generation configured (from src/CMakeLists.txt).")

# --- stellar-core Executable Definition ---
message(STATUS "  Defining stellar-core executable (from src/CMakeLists.txt)...")
set(STELLAR_CORE_MAIN_SOURCES_CTX "")
file(GLOB_RECURSE ALL_SRC_CPP_FILES_CTX "${CMAKE_CURRENT_SOURCE_DIR}/*.cpp")
set(EXCLUDE_PATTERNS_CTX
    "${CMAKE_CURRENT_SOURCE_DIR}/test/"
    "${CMAKE_CURRENT_SOURCE_DIR}/rust/"      # Rust sources are in librust_stellar_core.a
    "${CMAKE_CURRENT_SOURCE_DIR}/protocol-curr/" # XDR files are generated
    "${CMAKE_CURRENT_SOURCE_DIR}/protocol-next/" # XDR files are generated
    "${CMAKE_CURRENT_SOURCE_DIR}/history/serialize-tests/"
    "${CMAKE_CURRENT_SOURCE_DIR}/main/dumpxdr.cpp" # Separate tool
)
foreach(file_ctx ${ALL_SRC_CPP_FILES_CTX})
    set(exclude_file_ctx FALSE)
    foreach(pattern_ctx ${EXCLUDE_PATTERNS_CTX})
        if(file_ctx MATCHES "^${pattern_ctx}") set(exclude_file_ctx TRUE) break() endif()
    endforeach()
    if(NOT exclude_file_ctx) list(APPEND STELLAR_CORE_MAIN_SOURCES_CTX ${file_ctx}) endif()
endforeach()

list(APPEND STELLAR_CORE_MAIN_SOURCES_CTX ${GENERATED_CORE_SOURCES_SRC})
if(BUILD_TESTS) # BUILD_TESTS from parent scope
    file(GLOB_RECURSE TEST_SOURCES_CPP_CTX "${CMAKE_CURRENT_SOURCE_DIR}/test/*.cpp")
    if(TEST_SOURCES_CPP_CTX)
        list(APPEND STELLAR_CORE_MAIN_SOURCES_CTX ${TEST_SOURCES_CPP_CTX})
        message(STATUS "    Added test sources from src/test: ${TEST_SOURCES_CPP_CTX}")
    else()
        message(WARNING "    BUILD_TESTS is ON, but no .cpp sources found in src/test/.")
    endif()

    # Add simulation sources
    file(GLOB_RECURSE SIMULATION_SOURCES_CPP_CTX "${CMAKE_CURRENT_SOURCE_DIR}/simulation/*.cpp")
    if(SIMULATION_SOURCES_CPP_CTX)
        list(APPEND STELLAR_CORE_MAIN_SOURCES_CTX ${SIMULATION_SOURCES_CPP_CTX})
        message(STATUS "    Added simulation sources from src/simulation: ${SIMULATION_SOURCES_CPP_CTX}")
    else()
        message(WARNING "    BUILD_TESTS is ON, but no .cpp sources found in src/simulation/.")
    endif()
endif()

add_executable(stellar-core ${STELLAR_CORE_MAIN_SOURCES_CTX})

target_include_directories(stellar-core PUBLIC
    ${CMAKE_BINARY_DIR} # For config.h, and generated headers in subdirs like generated/xdr
    ${CMAKE_CURRENT_SOURCE_DIR}  # For headers within src/
)
if(BUILD_TESTS)
    target_include_directories(stellar-core PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/test
        ${CMAKE_CURRENT_SOURCE_DIR}/simulation # Add simulation include directory
    )
    message(STATUS "    Added src/test and src/simulation to stellar-core include directories.")
endif()

# Include directories from dependencies defined in root or lib/CMakeLists.txt
# These are typically handled by linking INTERFACE libraries.
# Explicit additions if variables are directly used (less common with modern CMake):
if(XDRPP_INCLUDE_DIRS AND NOT TARGET xdrpp) target_include_directories(stellar-core PUBLIC ${XDRPP_INCLUDE_DIRS}) endif()
if(LIBSODIUM_INCLUDE_DIRS AND NOT TARGET libsodium) target_include_directories(stellar-core PUBLIC ${LIBSODIUM_INCLUDE_DIRS}) endif()
if(POSTGRESQL_FOUND AND POSTGRESQL_INCLUDE_DIRS) target_include_directories(stellar-core PUBLIC ${POSTGRESQL_INCLUDE_DIRS}) endif()

target_link_libraries(stellar-core PRIVATE
    lib3rdparty_a       # Defined in lib/CMakeLists.txt
    soci_interface      # Defined in lib/CMakeLists.txt
    medida_interface    # Defined in lib/CMakeLists.txt or root
    xdrpp               # Defined in root
    libsodium           # Defined in root
    rust_stellar_core_lib # Defined in root
    fmt::fmt            # Defined in root (lib/fmt)
    cereal::cereal      # Defined in root (lib/cereal)
    Threads::Threads
)
if(ENABLE_TRACY AND TARGET TracyClient) target_link_libraries(stellar-core PRIVATE TracyClient) endif()
if(DL_LIBRARY) target_link_libraries(stellar-core PRIVATE ${DL_LIBRARY}) endif()
if(PLATFORM_SPECIFIC_LINK_LIBS) target_link_libraries(stellar-core PRIVATE ${PLATFORM_SPECIFIC_LINK_LIBS}) endif()

message(STATUS "  stellar-core executable configured (from src/CMakeLists.txt).")
message(STATUS "Finished processing src/CMakeLists.txt.")
