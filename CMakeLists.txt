cmake_minimum_required(VERSION 3.10)
project(njclient VERSION 1.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# -----------------------------------------------------------------------
# Find OGG and Vorbis (required)
# -----------------------------------------------------------------------
find_package(PkgConfig REQUIRED)
pkg_check_modules(OGG REQUIRED ogg)
pkg_check_modules(VORBIS REQUIRED vorbis)
pkg_check_modules(VORBISENC REQUIRED vorbisenc)

# -----------------------------------------------------------------------
# Source files
# -----------------------------------------------------------------------
set(NJCLIENT_SOURCES
    ninjam/njclient.cpp
    ninjam/netmsg.cpp
    ninjam/mpb.cpp
    WDL/sha.cpp
    WDL/rng.cpp
    WDL/win32_utf8.c
    WDL/jnetlib/connection.cpp
    WDL/jnetlib/asyncdns.cpp
    WDL/jnetlib/util.cpp
)

# win32_utf8.c uses wchar_t via wdlutf8.h; compile as C++ to get the type automatically
set_source_files_properties(WDL/win32_utf8.c PROPERTIES LANGUAGE CXX)

# -----------------------------------------------------------------------
# Static library target
# -----------------------------------------------------------------------
add_library(njclient STATIC ${NJCLIENT_SOURCES})

target_include_directories(njclient PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${OGG_INCLUDE_DIRS}
    ${VORBIS_INCLUDE_DIRS}
    ${VORBISENC_INCLUDE_DIRS}
)

target_compile_options(njclient PRIVATE
    ${OGG_CFLAGS_OTHER}
    ${VORBIS_CFLAGS_OTHER}
    ${VORBISENC_CFLAGS_OTHER}
)

if(UNIX AND NOT APPLE)
    target_link_libraries(njclient PUBLIC pthread)
endif()

# -----------------------------------------------------------------------
# Optional: install rules
# -----------------------------------------------------------------------
install(TARGETS njclient ARCHIVE DESTINATION lib)
install(FILES ninjam/njclient.h DESTINATION include/njclient)
