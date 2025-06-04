cmake_minimum_required(VERSION 3.16)

message(STATUS "Processing lib/CMakeLists.txt...")

# --- LibASIO ---
message(STATUS "  Configuring Asio from lib/CMakeLists.txt (Revised)...")
set(ASIO_CPP_FILE_LIB "${CMAKE_CURRENT_SOURCE_DIR}/asio.cpp") # Path relative to lib/
if(NOT EXISTS ${ASIO_CPP_FILE_LIB})
    message(FATAL_ERROR "Asio source file ${ASIO_CPP_FILE_LIB} not found in lib directory. Check submodule status or file path.")
endif()

add_library(asio_static_lib STATIC ${ASIO_CPP_FILE_LIB})
target_include_directories(asio_static_lib PRIVATE # PRIVATE, as these are for compiling asio.cpp itself
    ${CMAKE_CURRENT_SOURCE_DIR}/asio/asio/include
)
target_compile_definitions(asio_static_lib PRIVATE # PRIVATE, for compiling asio.cpp
    ASIO_SEPARATE_COMPILATION
    ASIO_STANDALONE
)

add_library(asio_interface INTERFACE)
target_include_directories(asio_interface INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/asio/asio/include  # Consumers need this path
)
target_compile_definitions(asio_interface INTERFACE
    ASIO_SEPARATE_COMPILATION  # Consumers need these defines
    ASIO_STANDALONE
)
target_link_libraries(asio_interface INTERFACE
    asio_static_lib # Consumers of asio_interface will also link to asio_static_lib
)
message(STATUS "  Asio configured from lib/CMakeLists.txt: asio_interface (links to asio_static_lib).")


# --- SOCI (database access library) ---
message(STATUS "  Configuring SOCI from lib/CMakeLists.txt (populating sources)...")
add_library(soci_lib STATIC "") # Initially empty, sources added below
add_library(soci_interface INTERFACE)

target_include_directories(soci_interface INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/soci/src/core
)
target_link_libraries(soci_lib PRIVATE soci_interface)
target_link_libraries(soci_interface INTERFACE soci_lib)

set(SOCI_CORE_SOURCES_LIB "")
file(GLOB SOCI_CORE_SOURCES_CPP_LIB "${CMAKE_CURRENT_SOURCE_DIR}/soci/src/core/*.cpp")
if(SOCI_CORE_SOURCES_CPP_LIB)
    list(APPEND SOCI_CORE_SOURCES_LIB ${SOCI_CORE_SOURCES_CPP_LIB})
    message(STATUS "    Adding SOCI core sources from lib: ${SOCI_CORE_SOURCES_CPP_LIB}")
else()
    message(WARNING "    No .cpp sources found in soci/src/core/ for soci_lib (from lib/CMakeLists.txt).")
endif()

if(SOCI_CORE_SOURCES_LIB)
    target_sources(soci_lib PRIVATE ${SOCI_CORE_SOURCES_LIB})
else()
    message(WARNING "    soci_lib has no core sources (from lib/CMakeLists.txt). This might be an error.")
endif()

# Conditional PostgreSQL backend for SOCI
# Options like DISABLE_POSTGRES and HAVE_POSTGRES are defined in the root CMakeLists.txt
if(NOT DISABLE_POSTGRES AND HAVE_POSTGRES)
    message(STATUS "    PostgreSQL is enabled for SOCI (from lib/CMakeLists.txt).")
    set(SOCI_POSTGRESQL_SOURCES_LIB "")
    file(GLOB SOCI_POSTGRESQL_SOURCES_CPP_LIB "${CMAKE_CURRENT_SOURCE_DIR}/soci/src/backends/postgresql/*.cpp")
    if(SOCI_POSTGRESQL_SOURCES_CPP_LIB)
        list(APPEND SOCI_POSTGRESQL_SOURCES_LIB ${SOCI_POSTGRESQL_SOURCES_CPP_LIB})
        target_sources(soci_lib PRIVATE ${SOCI_POSTGRESQL_SOURCES_LIB})
        message(STATUS "      Added SOCI PostgreSQL backend sources from lib: ${SOCI_POSTGRESQL_SOURCES_CPP_LIB}")
        # Link soci_lib to PostgreSQL::libpq (target PostgreSQL::libpq must be defined in root CMakeLists.txt)
        target_link_libraries(soci_lib PUBLIC PostgreSQL::libpq)
        message(STATUS "      soci_lib linked to PostgreSQL::libpq (from lib/CMakeLists.txt).")
    else()
        message(WARNING "    PostgreSQL backend for SOCI enabled, but no .cpp sources found in soci/src/backends/postgresql/ (from lib/CMakeLists.txt).")
    endif()
else()
    message(STATUS "    PostgreSQL backend for SOCI is disabled (from lib/CMakeLists.txt) (DISABLE_POSTGRES: ${DISABLE_POSTGRES}, HAVE_POSTGRES: ${HAVE_POSTGRES}).")
endif()
message(STATUS "  SOCI source population complete (from lib/CMakeLists.txt).")


# --- Libmedida (metrics library) ---
message(STATUS "  Configuring Libmedida from lib/CMakeLists.txt (populating sources if internal)...")
# Option USING_INTERNAL_LIBMEDIDA is defined in the root CMakeLists.txt
if(USING_INTERNAL_LIBMEDIDA)
    # Ensure medida_lib and medida_interface are defined if USING_INTERNAL_LIBMEDIDA was true in root
    # but the targets were not created there because pkg-config check happened first.
    # This logic assumes that if USING_INTERNAL_LIBMEDIDA is true, these targets should be fully defined here.
    if(NOT TARGET medida_lib)
      add_library(medida_lib STATIC "")
    endif()
    if(NOT TARGET medida_interface)
      add_library(medida_interface INTERFACE)
      target_include_directories(medida_interface INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/libmedida/src)
      target_link_libraries(medida_lib PRIVATE medida_interface)
      target_link_libraries(medida_interface INTERFACE medida_lib)
    endif()
    
    set(MEDIDA_INTERNAL_SOURCES_LIB "")
    file(GLOB MEDIDA_CORE_SOURCES_CPP_LIB "${CMAKE_CURRENT_SOURCE_DIR}/libmedida/src/medida/*.cpp")
    if(MEDIDA_CORE_SOURCES_CPP_LIB)
        list(APPEND MEDIDA_INTERNAL_SOURCES_LIB ${MEDIDA_CORE_SOURCES_CPP_LIB})
    endif()
    file(GLOB MEDIDA_REPORTING_SOURCES_CPP_LIB "${CMAKE_CURRENT_SOURCE_DIR}/libmedida/src/medida/reporting/*.cpp")
    if(MEDIDA_REPORTING_SOURCES_CPP_LIB)
        list(APPEND MEDIDA_INTERNAL_SOURCES_LIB ${MEDIDA_REPORTING_SOURCES_CPP_LIB})
    endif()
    file(GLOB MEDIDA_STATS_SOURCES_CPP_LIB "${CMAKE_CURRENT_SOURCE_DIR}/lib/libmedida/src/medida/stats/*.cpp") # Corrected path
    if(MEDIDA_STATS_SOURCES_CPP_LIB)
        list(APPEND MEDIDA_INTERNAL_SOURCES_LIB ${MEDIDA_STATS_SOURCES_CPP_LIB})
    endif()

    if(MEDIDA_INTERNAL_SOURCES_LIB)
        target_sources(medida_lib PRIVATE ${MEDIDA_INTERNAL_SOURCES_LIB})
        message(STATUS "      Added internal Libmedida sources from lib: ${MEDIDA_INTERNAL_SOURCES_LIB}")
    else()
        message(WARNING "      USING_INTERNAL_LIBMEDIDA is TRUE, but no sources found for libmedida in specified paths (from lib/CMakeLists.txt).")
    endif()
else()
     message(STATUS "    Using system/pkg-config libmedida (USING_INTERNAL_LIBMEDIDA: FALSE). medida_lib target not populated here.")
endif()
message(STATUS "  Libmedida configuration complete (from lib/CMakeLists.txt).")


# --- lib3rdparty_a (custom static library for various utilities) ---
message(STATUS "  Configuring lib3rdparty_a from lib/CMakeLists.txt...")
set(LIB3RDPARTY_SOURCES_LIB "")

file(GLOB UTIL_SOURCES_CPP_LIB "${CMAKE_CURRENT_SOURCE_DIR}/util/*.cpp")
if(UTIL_SOURCES_CPP_LIB)
    list(APPEND LIB3RDPARTY_SOURCES_LIB ${UTIL_SOURCES_CPP_LIB})
endif()

set(JSON_SOURCES_CPP_LIB "${CMAKE_CURRENT_SOURCE_DIR}/json/jsoncpp.cpp")
if(EXISTS ${JSON_SOURCES_CPP_LIB})
    list(APPEND LIB3RDPARTY_SOURCES_LIB ${JSON_SOURCES_CPP_LIB})
endif()

# Conditional sources for spdlog (if USE_SPDLOG and not built as separate target by add_subdirectory(lib/spdlog))
# This assumes spdlog_interface target is defined in root CMakeLists.txt
if(USE_SPDLOG AND NOT TARGET spdlog::spdlog) # Only add sources if spdlog isn't a separately built target
    file(GLOB SPDLOG_LIB_SOURCES_CPP_LIB "${CMAKE_CURRENT_SOURCE_DIR}/spdlog/src/*.cpp")
    list(FILTER SPDLOG_LIB_SOURCES_CPP_LIB EXCLUDE REGEX ".*bundled_fmtlib_format\\.cpp$")
    if(SPDLOG_LIB_SOURCES_CPP_LIB)
        list(APPEND LIB3RDPARTY_SOURCES_LIB ${SPDLOG_LIB_SOURCES_CPP_LIB})
        message(STATUS "    Added spdlog sources to lib3rdparty_a from lib: ${SPDLOG_LIB_SOURCES_CPP_LIB}")
    else()
        message(WARNING "    USE_SPDLOG is ON but no spdlog sources found for lib3rdparty_a in spdlog/src/ (from lib/CMakeLists.txt).")
    endif()
endif()

add_library(lib3rdparty_a STATIC ${LIB3RDPARTY_SOURCES_LIB})
target_include_directories(lib3rdparty_a PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/util
    ${CMAKE_CURRENT_SOURCE_DIR}/json
)
# Link lib3rdparty_a to other necessary libraries (targets defined in root or other subdirs)
target_link_libraries(lib3rdparty_a PUBLIC asio_interface sqlite3_lib)
if(USE_SPDLOG)
    target_link_libraries(lib3rdparty_a PUBLIC spdlog_interface) # spdlog_interface must be defined in root
endif()
message(STATUS "  lib3rdparty_a configuration complete (from lib/CMakeLists.txt).")

message(STATUS "Finished processing lib/CMakeLists.txt.")
