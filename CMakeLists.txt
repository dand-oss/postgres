# Ports build of PostgreSQL's frontend client library (libpq) for Windows x64
# with clang-cl / lld-link, built next to upstream's sources.
#
# Upstream sources are not modified. Generated files go to the build tree.
# The source lists follow src/interfaces/libpq/meson.build, src/common/meson.build
# and src/port/meson.build for host_system == 'windows' with OpenSSL enabled.
#
# Produces (static, Debug/Release):
#   libpq.lib            src/interfaces/libpq (incl. pthread-win32.c, win32.c, OpenSSL secure code)
#   pgcommon_shlib.lib   src/common frontend sources (shlib variant)
#   pgport_shlib.lib     src/port sources for Windows (shlib variant)
#
# Typical use:
#   cmake -G Ninja -S <this dir> -B <build> -DCMAKE_TOOLCHAIN_FILE=... \
#         -DOPENSSL_ROOT_DIR=<openssl prefix> -DCMAKE_INSTALL_PREFIX=<prefix>

cmake_minimum_required(VERSION 3.20)

project(postgres_libpq_win LANGUAGES C)

if(NOT CMAKE_SYSTEM_NAME STREQUAL "Windows")
    message(FATAL_ERROR "postgres_libpq_win targets Windows only (CMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME})")
endif()

set(PG_SRC "${CMAKE_CURRENT_SOURCE_DIR}")
set(PG_GEN "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(PG_GEN_INC "${PG_GEN}/src/include")
set(PG_GEN_COMMON "${PG_GEN}/src/common")

# ---------------------------------------------------------------------------
# Version: configure.ac is the single source of truth (meson reads it too).
# ---------------------------------------------------------------------------
file(STRINGS "${PG_SRC}/configure.ac" _pg_ac_init REGEX "^AC_INIT\\(\\[PostgreSQL\\], \\[[0-9.]+\\]")
if(NOT _pg_ac_init)
    message(FATAL_ERROR "cannot read PostgreSQL version from configure.ac")
endif()
string(REGEX REPLACE "^AC_INIT\\(\\[PostgreSQL\\], \\[([0-9.]+)\\].*" "\\1" PG_VERSION "${_pg_ac_init}")
string(REPLACE "." ";" _pg_version_parts "${PG_VERSION}")
list(GET _pg_version_parts 0 PG_MAJOR)
list(GET _pg_version_parts 1 PG_MINOR)
math(EXPR PG_VERSION_NUM "${PG_MAJOR} * 10000 + ${PG_MINOR}")
message(STATUS "PostgreSQL ${PG_VERSION} (PG_VERSION_NUM ${PG_VERSION_NUM})")

# ---------------------------------------------------------------------------
# Tools and OpenSSL
# ---------------------------------------------------------------------------
# errcodes.h and kwlist_d.h are produced by Perl generators in the source tree.
find_package(Perl REQUIRED)
find_package(OpenSSL REQUIRED)

# Windows system libraries libpq needs (meson: os_deps ws2_32, libpq_deps secur32).
add_library(pg_win_syslibs INTERFACE)
target_link_libraries(pg_win_syslibs INTERFACE
    ws2_32
    secur32
    advapi32
    crypt32
    shell32)

# ---------------------------------------------------------------------------
# Generated headers (build tree only)
# ---------------------------------------------------------------------------
file(MAKE_DIRECTORY "${PG_GEN_INC}/utils" "${PG_GEN_COMMON}")

configure_file("${PG_SRC}/cmake/pg_config.h.in" "${PG_GEN_INC}/pg_config.h" @ONLY)

set(PG_INT64_TYPE "long long int")
configure_file("${PG_SRC}/cmake/pg_config_ext.h.in" "${PG_GEN_INC}/pg_config_ext.h" @ONLY)

# pg_config_os.h is src/include/port/win32.h, copied as meson does.
configure_file("${PG_SRC}/src/include/port/win32.h" "${PG_GEN_INC}/pg_config_os.h" COPYONLY)

set(PG_PATH_BIN "${CMAKE_INSTALL_PREFIX}/bin")
set(PG_PATH_SHARE "${CMAKE_INSTALL_PREFIX}/share")
set(PG_PATH_SYSCONF "${CMAKE_INSTALL_PREFIX}/etc")
set(PG_PATH_INCLUDE "${CMAKE_INSTALL_PREFIX}/include")
set(PG_PATH_INCLUDE_SERVER "${CMAKE_INSTALL_PREFIX}/include/server")
set(PG_PATH_LIB "${CMAKE_INSTALL_PREFIX}/lib")
set(PG_PATH_LOCALE "${CMAKE_INSTALL_PREFIX}/share/locale")
set(PG_PATH_DOC "${CMAKE_INSTALL_PREFIX}/doc")
set(PG_PATH_MAN "${CMAKE_INSTALL_PREFIX}/share/man")
configure_file("${PG_SRC}/cmake/pg_config_paths.h.in" "${PG_GEN_INC}/pg_config_paths.h" @ONLY)

# errcodes.h (as src/include/utils/meson.build does)
add_custom_command(
    OUTPUT "${PG_GEN_INC}/utils/errcodes.h"
    COMMAND "${PERL_EXECUTABLE}" "${PG_SRC}/src/backend/utils/generate-errcodes.pl"
            --outfile "${PG_GEN_INC}/utils/errcodes.h"
            "${PG_SRC}/src/backend/utils/errcodes.txt"
    DEPENDS "${PG_SRC}/src/backend/utils/generate-errcodes.pl"
            "${PG_SRC}/src/backend/utils/errcodes.txt"
    COMMENT "Generating errcodes.h")

# kwlist_d.h (src/common/meson.build common_kwlist)
add_custom_command(
    OUTPUT "${PG_GEN_COMMON}/kwlist_d.h"
    COMMAND "${PERL_EXECUTABLE}" -I "${PG_SRC}/src/tools"
            "${PG_SRC}/src/tools/gen_keywordlist.pl"
            --output "${PG_GEN_COMMON}"
            "${PG_SRC}/src/include/parser/kwlist.h"
    DEPENDS "${PG_SRC}/src/tools/gen_keywordlist.pl"
            "${PG_SRC}/src/tools/PerfectHash.pm"
            "${PG_SRC}/src/include/parser/kwlist.h"
    COMMENT "Generating kwlist_d.h")

# Catalog headers (src/include/catalog/meson.build generated_catalog_headers).
# frontend code includes catalog/*_d.h (e.g. common/relpath.c), so the genbki
# outputs are generated into the build tree. The header list is read from
# meson.build so the two cannot drift apart.
file(READ "${PG_SRC}/src/include/catalog/meson.build" _pg_catalog_meson)
string(REGEX MATCH "catalog_headers = \\[[^]]*\\]" _pg_catalog_block "${_pg_catalog_meson}")
string(REGEX MATCHALL "'[a-z_]+\\.h'" _pg_catalog_names "${_pg_catalog_block}")
string(REPLACE "'" "" _pg_catalog_names "${_pg_catalog_names}")
if(NOT _pg_catalog_names)
    message(FATAL_ERROR "cannot read catalog_headers from src/include/catalog/meson.build")
endif()

set(PG_GEN_CATALOG "${PG_GEN_INC}/catalog")
file(MAKE_DIRECTORY "${PG_GEN_CATALOG}")
set(_pg_catalog_inputs "")
set(_pg_catalog_outputs
    "${PG_GEN_CATALOG}/postgres.bki"
    "${PG_GEN_CATALOG}/system_constraints.sql"
    "${PG_GEN_CATALOG}/schemapg.h"
    "${PG_GEN_CATALOG}/syscache_ids.h"
    "${PG_GEN_CATALOG}/syscache_info.h"
    "${PG_GEN_CATALOG}/system_fk_info.h")
foreach(_h IN LISTS _pg_catalog_names)
    list(APPEND _pg_catalog_inputs "${PG_SRC}/src/include/catalog/${_h}")
    string(REGEX REPLACE "\\.h$" "_d.h" _d "${_h}")
    list(APPEND _pg_catalog_outputs "${PG_GEN_CATALOG}/${_d}")
endforeach()

file(GLOB _pg_catalog_dat_files "${PG_SRC}/src/include/catalog/*.dat")
add_custom_command(
    OUTPUT ${_pg_catalog_outputs}
    COMMAND "${PERL_EXECUTABLE}" "${PG_SRC}/src/backend/catalog/genbki.pl"
            "--include-path=${PG_SRC}/src/include"
            "--set-version=${PG_MAJOR}"
            "--output=${PG_GEN_CATALOG}"
            ${_pg_catalog_inputs}
    DEPENDS ${_pg_catalog_inputs}
            ${_pg_catalog_dat_files}
            "${PG_SRC}/src/backend/catalog/genbki.pl"
            "${PG_SRC}/src/backend/catalog/Catalog.pm"
    COMMENT "Generating catalog headers (genbki)")

add_custom_target(pg_generated_headers
    DEPENDS
        "${PG_GEN_INC}/utils/errcodes.h"
        "${PG_GEN_COMMON}/kwlist_d.h"
        ${_pg_catalog_outputs})

# ---------------------------------------------------------------------------
# Shared compile settings (meson: frontend_*_code and cppflags for MSVC)
# ---------------------------------------------------------------------------
set(PG_FE_INCLUDE_DIRS
    "${PG_SRC}/src/include"
    "${PG_SRC}/src/include/port/win32"
    "${PG_SRC}/src/include/port/win32_msvc"
    "${PG_GEN_INC}")

set(PG_FE_DEFINES
    FRONTEND
    WIN32
    WINDOWS
    __WINDOWS__
    __WIN32__
    _CRT_SECURE_NO_DEPRECATE
    _CRT_NONSTDC_NO_DEPRECATE)

# ---------------------------------------------------------------------------
# pgport_shlib: src/port (meson src/port/meson.build, host_system == 'windows')
# ---------------------------------------------------------------------------
set(PG_PORT_SOURCES
    bsearch_arg.c
    chklocale.c
    inet_net_ntop.c
    noblock.c
    path.c
    pg_bitutils.c
    pg_strong_random.c
    pgcheckdir.c
    pgmkdirp.c
    pgsleep.c
    pgstrcasecmp.c
    pgstrsignal.c
    pqsignal.c
    qsort.c
    qsort_arg.c
    quotes.c
    snprintf.c
    strerror.c
    tar.c
    user.c
    # windows
    dirmod.c
    kill.c
    open.c
    system.c
    win32common.c
    win32dlopen.c
    win32env.c
    win32error.c
    win32fdatasync.c
    win32fseek.c
    win32gai_strerror.c
    win32getrusage.c
    win32link.c
    win32ntdll.c
    win32pread.c
    win32pwrite.c
    win32security.c
    win32setlocale.c
    win32stat.c
    # cc.get_id() == 'msvc' (clang-cl uses the MSVC ABI and headers)
    dirent.c
    win32gettimeofday.c
    # replace_funcs_neg: built because HAVE_* is undefined on MSVC
    explicit_bzero.c
    getopt.c
    getopt_long.c
    getpeereid.c
    inet_aton.c
    mkdtemp.c
    strlcat.c
    strlcpy.c
    timingsafe_bcmp.c
    # replace_funcs_pos: runtime-checked SSE4.2 CRC32C (USE_SSE42_CRC32C_WITH_RUNTIME_CHECK)
    pg_crc32c_sse42_choose.c
    pg_crc32c_sb8.c)

set(PG_PORT_SRC_FILES "")
foreach(_f IN LISTS PG_PORT_SOURCES)
    list(APPEND PG_PORT_SRC_FILES "${PG_SRC}/src/port/${_f}")
endforeach()
# pg_crc32c_sse42.c is built with cflags_crc in meson's non-MSVC branch. On
# clang-cl the CRC32 intrinsics need the sse4.2 target feature enabled per unit,
# so pass -msse4.2 to this one source (runtime CPUID check still selects it).
list(APPEND PG_PORT_SRC_FILES "${PG_SRC}/src/port/pg_crc32c_sse42.c")
set_source_files_properties("${PG_SRC}/src/port/pg_crc32c_sse42.c" PROPERTIES COMPILE_OPTIONS "-msse4.2")

add_library(pgport_shlib STATIC ${PG_PORT_SRC_FILES})
target_include_directories(pgport_shlib PRIVATE ${PG_FE_INCLUDE_DIRS})
target_compile_definitions(pgport_shlib PRIVATE ${PG_FE_DEFINES})
# meson: pgport objects depend on ssl (pg_strong_random.c, timingsafe_bcmp.c)
target_link_libraries(pgport_shlib PRIVATE OpenSSL::SSL OpenSSL::Crypto)

# ---------------------------------------------------------------------------
# pgcommon_shlib: src/common (meson src/common/meson.build, frontend shlib variant)
# ---------------------------------------------------------------------------
set(PG_COMMON_SOURCES
    archive.c
    base64.c
    binaryheap.c
    blkreftable.c
    checksum_helper.c
    compression.c
    controldata_utils.c
    encnames.c
    exec.c
    file_perm.c
    file_utils.c
    hashfn.c
    ip.c
    jsonapi.c
    keywords.c
    kwlookup.c
    link-canary.c
    md5_common.c
    parse_manifest.c
    percentrepl.c
    pg_get_line.c
    pg_lzcompress.c
    pg_prng.c
    pgfnames.c
    psprintf.c
    relpath.c
    rmtree.c
    saslprep.c
    scram-common.c
    string.c
    stringinfo.c
    unicode_case.c
    unicode_category.c
    unicode_norm.c
    username.c
    wait_error.c
    wchar.c
    # ssl.found()
    cryptohash_openssl.c
    hmac_openssl.c
    protocol_openssl.c
    # common_sources_frontend_shlib additions (logging.c is excluded for shlib)
    fe_memutils.c
    restricted_token.c
    sprompt.c
    # common_sources_cflags: ryu and config_info (link_whole into pgcommon)
    d2s.c
    f2s.c
    config_info.c)

set(PG_COMMON_SRC_FILES "")
foreach(_f IN LISTS PG_COMMON_SOURCES)
    list(APPEND PG_COMMON_SRC_FILES "${PG_SRC}/src/common/${_f}")
endforeach()

# config_info.c takes its VAL_* strings from the meson configuration. Values
# here are the MSVC-style equivalents; pg_config --configure output is not
# used by libpq.
set_source_files_properties("${PG_SRC}/src/common/config_info.c" PROPERTIES COMPILE_DEFINITIONS
    "VAL_CC=\"clang-cl\";VAL_CPPFLAGS=\"\";VAL_CFLAGS=\"\";VAL_CFLAGS_SL=\"\";VAL_LDFLAGS=\"\";VAL_LDFLAGS_EX=\"\";VAL_LDFLAGS_SL=\"\";VAL_LIBS=\"\"")

add_library(pgcommon_shlib STATIC ${PG_COMMON_SRC_FILES})
target_include_directories(pgcommon_shlib PRIVATE
    ${PG_FE_INCLUDE_DIRS}
    "${PG_SRC}/src/common"
    "${PG_GEN_COMMON}")
target_compile_definitions(pgcommon_shlib PRIVATE ${PG_FE_DEFINES})
target_link_libraries(pgcommon_shlib PRIVATE OpenSSL::SSL OpenSSL::Crypto)
add_dependencies(pgcommon_shlib pg_generated_headers)

add_dependencies(pgport_shlib pg_generated_headers)

# ---------------------------------------------------------------------------
# libpq (static): src/interfaces/libpq/meson.build libpq_sources
# ---------------------------------------------------------------------------
set(PG_LIBPQ_SOURCES
    fe-auth-scram.c
    fe-auth.c
    fe-cancel.c
    fe-connect.c
    fe-exec.c
    fe-lobj.c
    fe-misc.c
    fe-print.c
    fe-protocol3.c
    fe-secure.c
    fe-trace.c
    legacy-pqsignal.c
    libpq-events.c
    pqexpbuffer.c
    # windows
    pthread-win32.c
    win32.c
    # ssl.found()
    fe-secure-common.c
    fe-secure-openssl.c)

set(PG_LIBPQ_SRC_FILES "")
foreach(_f IN LISTS PG_LIBPQ_SOURCES)
    list(APPEND PG_LIBPQ_SRC_FILES "${PG_SRC}/src/interfaces/libpq/${_f}")
endforeach()

add_library(libpq STATIC ${PG_LIBPQ_SRC_FILES})
target_include_directories(libpq PRIVATE
    ${PG_FE_INCLUDE_DIRS}
    "${PG_SRC}/src/interfaces/libpq"
    "${PG_SRC}/src/port")
target_compile_definitions(libpq PRIVATE ${PG_FE_DEFINES} SO_MAJOR_VERSION=5)
target_link_libraries(libpq
    PUBLIC
        pgcommon_shlib
        pgport_shlib
        OpenSSL::SSL
        OpenSSL::Crypto
        pg_win_syslibs)
add_dependencies(libpq pg_generated_headers)

# ---------------------------------------------------------------------------
# Install: headers that find_package(PostgreSQL) and consumers need, and libs
# ---------------------------------------------------------------------------
install(FILES
    "${PG_SRC}/src/interfaces/libpq/libpq-fe.h"
    "${PG_SRC}/src/interfaces/libpq/libpq-events.h"
    "${PG_SRC}/src/include/postgres_ext.h"
    "${PG_GEN_INC}/pg_config.h"
    "${PG_GEN_INC}/pg_config_ext.h"
    "${PG_GEN_INC}/pg_config_os.h"
    DESTINATION include)

install(TARGETS libpq pgcommon_shlib pgport_shlib
    ARCHIVE DESTINATION lib)
