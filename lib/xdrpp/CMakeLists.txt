cmake_minimum_required(VERSION 3.16)
project(xdrpp_project LANGUAGES CXX C) # Enable C for getopt_long.c

message(STATUS "Processing lib/xdrpp/CMakeLists.txt...")

# Configure build_endian.h
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/xdrpp/build_endian.h.in"
    "${CMAKE_CURRENT_BINARY_DIR}/xdrpp/build_endian.h"
)
message(STATUS "  Configured xdrpp/build_endian.h")

# Define xdrpp_lib (Static Library)
add_library(xdrpp_lib STATIC)
message(STATUS "  Defined static library target: xdrpp_lib")

file(GLOB XDRPP_LIB_SOURCES
    RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}"
    "xdrpp/*.cc"
    "compat/getopt_long.c" # Added C source
)
# Headers are not added via target_sources for compilation usually, but glob them for verification if needed
file(GLOB XDRPP_LIB_HEADERS RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "xdrpp/*.h")
message(STATUS "  Globbed xdrpp_lib sources: ${XDRPP_LIB_SOURCES}")
message(STATUS "  Globbed xdrpp_lib headers (for info): ${XDRPP_LIB_HEADERS}")


target_sources(xdrpp_lib PRIVATE ${XDRPP_LIB_SOURCES})

target_include_directories(xdrpp_lib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}> # For xdrpp/*.h within the build
        $<INSTALL_INTERFACE:include> # For xdrpp/*.h when installed (see install rule)
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}> # For build_endian.h
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/compat" # For getopt.h used by getopt_long.c
)
message(STATUS "  Set include directories for xdrpp_lib.")

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

file(GLOB XDRC_OTHER_SOURCES
    RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}"
    "xdrc/*.cc"
    "xdrc/*.h" # For info, not compilation typically
)
# Exclude generated files from this glob if they have same names as source files (e.g. if scan.cc was in source tree)
list(FILTER XDRC_OTHER_SOURCES EXCLUDE REGEX ".*scan\\.cc$")
list(FILTER XDRC_OTHER_SOURCES EXCLUDE REGEX ".*parse\\.cc$")

list(APPEND XDRC_SOURCES ${XDRC_OTHER_SOURCES} ${XDRC_GENERATED_SOURCES})
message(STATUS "  Globbed/Generated xdrc_executable sources: ${XDRC_SOURCES}")

add_executable(xdrc_executable ${XDRC_SOURCES})

target_include_directories(xdrc_executable PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR} # For xdrc_internal.h, union.h, etc.
    ${CMAKE_CURRENT_BINARY_DIR} # For generated parse.h
    ${CMAKE_CURRENT_BINARY_DIR}/xdrpp # For build_endian.h (if xdrc sources need it)
)
target_link_libraries(xdrc_executable PRIVATE xdrpp_lib)
message(STATUS "  Defined executable target: xdrc_executable, linked to xdrpp_lib.")

# Installation rules
if(CMAKE_INSTALL_LIBDIR) # Ensure GNUInstallDirs has set this
    install(TARGETS xdrpp_lib 
        EXPORT xdrpp_targets # Export for find_package
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
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/xdrpp"
    )
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
