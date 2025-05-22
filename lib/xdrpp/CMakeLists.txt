cmake_minimum_required(VERSION 3.16)
project(xdrpp_project LANGUAGES CXX C) # Enable C for getopt_long.c

include(CheckFunctionExists) # For getopt_long_only check

message(STATUS "Processing lib/xdrpp/CMakeLists.txt...")

# Check for getopt_long_only
check_function_exists(getopt_long_only HAVE_GETOPT_LONG_ONLY)
if(HAVE_GETOPT_LONG_ONLY)
    message(STATUS "  Found getopt_long_only in system libraries.")
else()
    message(STATUS "  getopt_long_only not found in system libraries. Will use compat/getopt_long.c and define NEED_GETOPT_LONG.")
endif()

# Fix Endianness for build_endian.h.in
include(TestBigEndian)
TEST_BIG_ENDIAN(LOCAL_CMAKE_IS_BIG_ENDIAN_RESULT) # Use a local variable to avoid potential clashes
if(LOCAL_CMAKE_IS_BIG_ENDIAN_RESULT)
    set(WORDS_BIGENDIAN 1 CACHE INTERNAL "System is big endian for xdrpp")
    message(STATUS "  xdrpp: System is BIG ENDIAN.")
else()
    # If not big endian, WORDS_BIGENDIAN should not be defined or be 0 for the .in file to work as expected
    # (i.e., /* #undef WORDS_BIGENDIAN */ remains if it's little endian)
    # So, we don't set WORDS_BIGENDIAN to 0, we just don't define it to 1.
    # However, if build_endian.h.in specifically uses #cmakedefine01, then set to 0.
    # Given it's /* #undef WORDS_BIGENDIAN */, if WORDS_BIGENDIAN is not 1, it stays undefined.
    # For clarity, let's ensure it's explicitly handled if the .in file changes.
    # The .in file expects WORDS_BIGENDIAN to be defined (or not) for /* #undef WORDS_BIGENDIAN */.
    # If it were #cmakedefine WORDS_BIGENDIAN, then setting it to 0 or 1 would be appropriate.
    # Let's assume the current .in file: /* #undef WORDS_BIGENDIAN */
    # If big endian, CMake variable WORDS_BIGENDIAN=1 -> #define WORDS_BIGENDIAN 1
    # If little endian, CMake variable WORDS_BIGENDIAN not set or set to 0/OFF -> /* #undef WORDS_BIGENDIAN */
    # The TestBigEndian module sets CMAKE_IS_BIG_ENDIAN_RESULT to TRUE or FALSE.
    if(LOCAL_CMAKE_IS_BIG_ENDIAN_RESULT)
      set(WORDS_BIGENDIAN 1)
    else()
      # Do not define WORDS_BIGENDIAN or set to 0 if #cmakedefine is used.
      # For /* #undef WORDS_BIGENDIAN */, not defining it or setting to 0/OFF/FALSE works.
      unset(WORDS_BIGENDIAN) # Ensure it's not defined if little endian for the /* #undef */ pattern
    endif()
    message(STATUS "  xdrpp: System is LITTLE ENDIAN (WORDS_BIGENDIAN not set to 1).")
endif()

# Configure build_endian.h
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/xdrpp/build_endian.h.in"
    "${CMAKE_CURRENT_BINARY_DIR}/xdrpp/build_endian.h"
    # No @ONLY, to allow #cmakedefine or /* #undef */ processing based on WORDS_BIGENDIAN variable
)
message(STATUS "  Configured xdrpp/build_endian.h (CMake variable WORDS_BIGENDIAN value: ${WORDS_BIGENDIAN})")


# Define xdrpp_lib (Static Library)
add_library(xdrpp_lib STATIC)
message(STATUS "  Defined static library target: xdrpp_lib")

set(XDRPP_LIB_SOURCES_INIT "")
file(GLOB XDRPP_CC_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "xdrpp/*.cc")
list(APPEND XDRPP_LIB_SOURCES_INIT ${XDRPP_CC_FILES})

if(NOT HAVE_GETOPT_LONG_ONLY)
    list(APPEND XDRPP_LIB_SOURCES_INIT "compat/getopt_long.c")
endif()

# Headers are not added via target_sources for compilation usually, but glob them for verification if needed
file(GLOB XDRPP_LIB_HEADERS RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "xdrpp/*.h")
message(STATUS "  Collected xdrpp_lib sources: ${XDRPP_LIB_SOURCES_INIT}")
message(STATUS "  Globbed xdrpp_lib headers (for info): ${XDRPP_LIB_HEADERS}")

target_sources(xdrpp_lib PRIVATE ${XDRPP_LIB_SOURCES_INIT})

target_include_directories(xdrpp_lib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}> # For xdrpp/*.h within the build
        $<INSTALL_INTERFACE:include> # For xdrpp/*.h when installed (see install rule)
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}> # For build_endian.h
    PRIVATE
        # No longer always add compat, will add conditionally below
)
if(NOT HAVE_GETOPT_LONG_ONLY)
    target_include_directories(xdrpp_lib PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/compat")
    target_compile_definitions(xdrpp_lib PRIVATE NEED_GETOPT_LONG)
    message(STATUS "  xdrpp_lib: Added compat/ include and NEED_GETOPT_LONG definition.")
endif()
# Defer message for include directories until all are added.

# Link xdrpp_lib against Cereal (if available as a target from root CMakeLists.txt)
if(TARGET cereal::cereal)
    target_link_libraries(xdrpp_lib PUBLIC cereal::cereal)
    message(STATUS "  xdrpp_lib: Linked PUBLIC to cereal::cereal.")
else()
    message(WARNING "  xdrpp_lib: Target cereal::cereal not found. Cannot link xdrpp_lib against Cereal. This might be an issue if xdrpp sources use Cereal.")
endif()

# Define xdrc_executable
message(STATUS "  Configuring xdrc_executable...")
find_package(FLEX QUIET)
find_package(BISON QUIET)

set(XDRC_SOURCES "")
set(XDRC_GENERATED_SOURCES "")

set(XDRC_LEXER_INPUT "${CMAKE_CURRENT_SOURCE_DIR}/xdrc/scan.ll")
set(XDRC_LEXER_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/xdrc/scan.cc") # Output to build directory
if(FLEX_FOUND)
    message(STATUS "    Found Flex: ${FLEX_EXECUTABLE}")
    add_custom_command(
        OUTPUT ${XDRC_LEXER_OUTPUT}
        COMMAND ${FLEX_EXECUTABLE} --outfile=${XDRC_LEXER_OUTPUT} ${XDRC_LEXER_INPUT}
        DEPENDS ${XDRC_LEXER_INPUT}
        COMMENT "Generating C++ scanner from scan.ll for xdrc"
    )
    list(APPEND XDRC_GENERATED_SOURCES ${XDRC_LEXER_OUTPUT})
else()
    message(WARNING "    Flex not found. xdrc scanner (scan.cc) cannot be generated.")
endif()

set(XDRC_PARSER_INPUT "${CMAKE_CURRENT_SOURCE_DIR}/xdrc/parse.yy")
set(XDRC_PARSER_OUTPUT_CPP "${CMAKE_CURRENT_BINARY_DIR}/xdrc/parse.cc") # Output to build directory
set(XDRC_PARSER_OUTPUT_H "${CMAKE_CURRENT_BINARY_DIR}/xdrc/parse.h")   # Output to build directory
if(BISON_FOUND)
    message(STATUS "    Found Bison: ${BISON_EXECUTABLE}")
    add_custom_command(
        OUTPUT ${XDRC_PARSER_OUTPUT_CPP} ${XDRC_PARSER_OUTPUT_H}
        COMMAND ${BISON_EXECUTABLE} --defines=${XDRC_PARSER_OUTPUT_H} --output=${XDRC_PARSER_OUTPUT_CPP} ${XDRC_PARSER_INPUT}
        DEPENDS ${XDRC_PARSER_INPUT}
        COMMENT "Generating C++ parser from parse.yy for xdrc"
    )
    list(APPEND XDRC_GENERATED_SOURCES ${XDRC_PARSER_OUTPUT_CPP})
else()
    message(WARNING "    Bison not found. xdrc parser (parse.cc/h) cannot be generated.")
endif()

file(GLOB XDRC_CC_FILES
    RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}"
    "xdrc/*.cc"
)
# Exclude generated files from this glob if they have same names as source files
list(FILTER XDRC_CC_FILES EXCLUDE REGEX ".*scan\\.cc$") # if scan.cc could exist in source
list(FILTER XDRC_CC_FILES EXCLUDE REGEX ".*parse\\.cc$")# if parse.cc could exist in source

set(XDRC_EXEC_SOURCES_INIT "")
list(APPEND XDRC_EXEC_SOURCES_INIT ${XDRC_CC_FILES} ${XDRC_GENERATED_SOURCES})

if(NOT HAVE_GETOPT_LONG_ONLY)
    list(APPEND XDRC_EXEC_SOURCES_INIT "compat/getopt_long.c")
endif()

message(STATUS "  Collected/Generated xdrc_executable sources: ${XDRC_EXEC_SOURCES_INIT}")

add_executable(xdrc_executable ${XDRC_EXEC_SOURCES_INIT})

target_include_directories(xdrc_executable PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR} # For xdrc_internal.h, union.h, etc.
    ${CMAKE_CURRENT_BINARY_DIR} # For generated parse.h
    ${CMAKE_CURRENT_BINARY_DIR}/xdrpp # For build_endian.h (if xdrc sources need it)
)
if(NOT HAVE_GETOPT_LONG_ONLY)
    target_include_directories(xdrc_executable PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/compat")
    target_compile_definitions(xdrc_executable PRIVATE NEED_GETOPT_LONG)
    message(STATUS "  xdrc_executable: Added compat/ include and NEED_GETOPT_LONG definition.")
endif()
target_link_libraries(xdrc_executable PRIVATE xdrpp_lib) # xdrc needs to link against xdrpp_lib
message(STATUS "  Defined executable target: xdrc_executable, linked to xdrpp_lib.")

# Add Internal XDR Generation for rpc_msg.x and rpcb_prot.x
set(XDRPP_INTERNAL_X_FILES rpc_msg.x rpcb_prot.x)
set(XDRPP_GENERATED_INTERNAL_HEADERS_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated_xdr_headers_internal") # Changed from previous attempt to be more specific
file(MAKE_DIRECTORY ${XDRPP_GENERATED_INTERNAL_HEADERS_DIR})
message(STATUS "  Internal XDR generated headers will be in: ${XDRPP_GENERATED_INTERNAL_HEADERS_DIR}")

set(XDRPP_GENERATED_HH_FILES "") # Initialize list for generated .hh files
foreach(X_FILE_BASENAME ${XDRPP_INTERNAL_X_FILES})
    set(INPUT_X_FILE "${CMAKE_CURRENT_SOURCE_DIR}/xdrpp/${X_FILE_BASENAME}")
    set(OUTPUT_HH_FILE "${XDRPP_GENERATED_INTERNAL_HEADERS_DIR}/${X_FILE_BASENAME}.hh")

    if(NOT EXISTS ${INPUT_X_FILE})
        message(FATAL_ERROR "Internal XDR source file not found: ${INPUT_X_FILE}")
        continue()
    endif()

    if(NOT TARGET xdrc_executable) # Ensure xdrc_executable target is defined
        message(FATAL_ERROR "Target xdrc_executable not found before trying to use it for internal XDR generation.")
    endif()

    add_custom_command(
        OUTPUT ${OUTPUT_HH_FILE}
        COMMAND $<TARGET_FILE:xdrc_executable> -hh -o ${OUTPUT_HH_FILE} ${INPUT_X_FILE}
        DEPENDS ${INPUT_X_FILE} xdrc_executable # Depends on the xdrc_executable target
        COMMENT "Generating ${OUTPUT_HH_FILE} from ${INPUT_X_FILE} for xdrpp_lib"
    )
    list(APPEND XDRPP_GENERATED_HH_FILES ${OUTPUT_HH_FILE})
    message(STATUS "    Added custom command for internal XDR: ${INPUT_X_FILE} -> ${OUTPUT_HH_FILE}")
endforeach()

# Add generated headers directory to include path for xdrpp_lib
target_include_directories(xdrpp_lib PUBLIC
    $<BUILD_INTERFACE:${XDRPP_GENERATED_INTERNAL_HEADERS_DIR}> # For build tree usage
    # For install tree, these will be installed to <prefix>/include/xdrpp/ (flat)
)
# xdrc_executable itself does not need to include these generated headers (rpc_msg.x.hh etc.)
# So, no need to add XDRPP_GENERATED_INTERNAL_HEADERS_DIR to xdrc_executable's includes.

message(STATUS "  Set include directories for xdrpp_lib (including generated XDR).")


# Installation rules
if(CMAKE_INSTALL_LIBDIR) # Ensure GNUInstallDirs has set this in root
    install(TARGETS xdrpp_lib
        EXPORT xdrpp_project_targets # Changed export name to be unique
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} # For shared libs if any
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} # For DLLs on Windows
    )
    message(STATUS "    Added install rule for xdrpp_lib.")
else()
    message(WARNING "    CMAKE_INSTALL_LIBDIR not defined. Skipping install rule for xdrpp_lib.")
endif()

if(CMAKE_INSTALL_BINDIR)
    install(TARGETS xdrc_executable RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
    message(STATUS "    Added install rule for xdrc_executable.")
else()
    message(WARNING "    CMAKE_INSTALL_BINDIR not defined. Skipping install rule for xdrc_executable.")
endif()

if(CMAKE_INSTALL_INCLUDEDIR)
    install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/xdrpp/" # Install contents of xdrpp/
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/xdrpp"
        FILES_MATCHING PATTERN "*.h"
    )
    # Install configured build_endian.h
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/xdrpp/build_endian.h"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/xdrpp" # Installs to <prefix>/include/xdrpp/build_endian.h
    )
    # Install internally generated XDR headers
    if(XDRPP_GENERATED_HH_FILES)
        install(FILES ${XDRPP_GENERATED_HH_FILES}
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/xdrpp" 
            # Install them flat into xdrpp/ directory alongside other .h files.
        )
        message(STATUS "    Added install rule for internally generated XDR headers to ${CMAKE_INSTALL_INCLUDEDIR}/xdrpp.")
    endif()
    message(STATUS "    Added install rule for xdrpp headers.")
else()
    message(WARNING "    CMAKE_INSTALL_INCLUDEDIR not defined. Skipping install rule for xdrpp headers.")
endif()

# Export targets for find_package
install(EXPORT xdrpp_targets
  FILE xdrpp-targets.cmake
  NAMESPACE xdrpp::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/xdrpp
)
message(STATUS "  Added install(EXPORT ...) for xdrpp_targets.")

message(STATUS "Finished processing lib/xdrpp/CMakeLists.txt.")
