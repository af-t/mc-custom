cmake_minimum_required(VERSION 3.16)

set(MC_NAME "GNU Midnight Commander")
set(MC_PACKAGE "mc")
set(MC_URL "https://midnight-commander.org")

project(mc
    VERSION 4.8.32
    DESCRIPTION "GNU Midnight Commander"
    HOMEPAGE_URL ${MC_URL}
    LANGUAGES C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)

# Equivalent of autoconf's AC_USE_SYSTEM_EXTENSIONS: define _GNU_SOURCE on
# glibc so that GNU extensions (sighandler_t, statlstat, major(), ...) are
# visible consistently in both the config probes and the real translation
# units.
add_compile_definitions(_GNU_SOURCE)

# ---------------------------------------------------------------------------
# Version handling.  The autotools version.sh derives the version from git
# tags into mc-version.h.  Replicate that: use the current git describe, or
# fall back to the project version.
# ---------------------------------------------------------------------------
find_package(Git QUIET)
set(MC_CURRENT_VERSION "${PROJECT_VERSION}")
if(GIT_FOUND AND EXISTS "${CMAKE_SOURCE_DIR}/.git")
    execute_process(
        COMMAND ${GIT_EXECUTABLE} -C ${CMAKE_SOURCE_DIR} describe --always
        OUTPUT_VARIABLE _git_describe
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET)
    if(_git_describe)
        set(MC_CURRENT_VERSION "${_git_describe}")
    endif()
endif()

configure_file(${CMAKE_SOURCE_DIR}/cmake/mc-version.h.in
    ${CMAKE_BINARY_DIR}/mc-version.h @ONLY)

add_library(mc-version INTERFACE)
target_include_directories(mc-version INTERFACE
    ${CMAKE_BINARY_DIR}
    ${CMAKE_SOURCE_DIR})

# ---------------------------------------------------------------------------
# Honor CC=, CXX=, CFLAGS=, LDFLAGS= passed on the CMake command line.
# CMake honors CC/CXX via the toolchain and CFLAGS via its own variables, but
# we make sure a cache-level override on the command line is respected.
# ---------------------------------------------------------------------------
if(DEFINED ENV{CFLAGS} AND NOT DEFINED CMAKE_C_FLAGS)
    set(CMAKE_C_FLAGS "$ENV{CFLAGS}" CACHE STRING "Flags for C compiler" FORCE)
endif()
if(DEFINED ENV{LDFLAGS} AND NOT DEFINED CMAKE_EXE_LINKER_FLAGS)
    set(CMAKE_EXE_LINKER_FLAGS "$ENV{LDFLAGS}" CACHE STRING "Flags for linking executables" FORCE)
endif()
if(DEFINED ENV{LDFLAGS} AND NOT DEFINED CMAKE_SHARED_LINKER_FLAGS)
    set(CMAKE_SHARED_LINKER_FLAGS "$ENV{LDFLAGS}" CACHE STRING "Flags for linking shared libraries" FORCE)
endif()

include(GNUInstallDirs)
include(CheckCCompilerFlag)
include(CheckIncludeFile)
include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckTypeSize)
include(CheckCSourceCompiles)
include(CheckSymbolExists)
include(CheckStructHasMember)
include(CheckCSourceRuns)
include(CheckLibraryExists)

if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    foreach(_flag
        -Wall -Wextra
        -Wattributes -Wimplicit-function-declaration -Wimplicit-int
        -Wno-declaration-after-statement -Wno-long-long -Wno-vla
        -Wincompatible-pointer-types -Wint-conversion
        -Wbad-function-cast -Wfloat-conversion -Wfloat-equal
        -Wformat-security -Wimplicit-fallthrough -Wmissing-declarations
        -Wmissing-format-attribute -Wmissing-prototypes -Wpointer-arith
        -Wredundant-decls -Wshadow -Wstrict-prototypes -Wswitch-default
        -Wundef -Wunreachable-code -Wunused-result -Wwrite-strings
        -Wno-cast-function-type -Wno-assign-enum)
        check_c_compiler_flag(${_flag} _mc_flag_ok)
        if(_mc_flag_ok)
    set(MC_COMPILE_OPTIONS ${MC_COMPILE_OPTIONS} ${_flag})
endif()
    endforeach()
endif()

if(CMAKE_C_COMPILER_ID MATCHES "GNU")
    check_c_compiler_flag(-fdiagnostics-show-option _mc_flag_ok)
    if(_mc_flag_ok)
    set(MC_COMPILE_OPTIONS ${MC_COMPILE_OPTIONS} -fdiagnostics-show-option)
endif()
endif()

option(MC_ENABLE_WERROR "Treat all compiler warnings as errors" OFF)
if(MC_ENABLE_WERROR)
    set(MC_COMPILE_OPTIONS ${MC_COMPILE_OPTIONS} -Werror)
endif()

option(MC_WITH_GPM "PCRE-free xterm only mouse. gpm support on Linux" ON)

# ---------------------------------------------------------------------------
# Feature options (mirror the autotools defaults)
# ---------------------------------------------------------------------------
option(MC_ENABLE_ASSERT "Enable assertions" ON)
option(MC_MAINTAINER_MODE "Enable maintainer mode (debug build, extra sources)" OFF)
option(MC_ENABLE_BACKGROUND "Support for background file operations" ON)
option(MC_ENABLE_SUBSHELL "Compile in concurrent subshell" ON)
set(MC_SUBSHELL_OPTIONAL "" CACHE STRING "If set 'ON', make subshell support optional")
option(MC_WITH_INTERNAL_EDIT "Enable internal editor" ON)
option(MC_WITH_DIFF_VIEWER "Compile with the diff viewer" ON)
option(MC_WITH_VFS "Enable virtual filesystem support" ON)
option(MC_WITH_VFS_EXTFS "Support the extfs filesystem" ON)
option(MC_WITH_VFS_SFS "Support the sfs filesystem" ON)
option(MC_WITH_VFS_FTP "Support the FTP filesystem" ON)
option(MC_WITH_VFS_SHELL "Support the SHELL filesystem" ON)
option(MC_WITH_VFS_TAR "Support the tar filesystem" ON)
option(MC_WITH_VFS_CPIO "Support the cpio filesystem" ON)
option(MC_WITH_VFS_SFTP "Support the SFTP filesystem (requires libssh2)" ON)
option(MC_ENABLE_EXT2FS_ATTR "Support ext2/3/4 attributes (auto)" ON)
option(MC_ENABLE_CONFIGURE_ARGS "Embed configure arguments in help" OFF)
option(MC_ENABLE_TESTS "Enable unit tests (requires Check)" OFF)
set(MC_SCREEN_LIBRARY "auto" CACHE STRING "Screen library: auto, ncurses, slang")

# ---------------------------------------------------------------------------
# pkg-config based dependency detection
# ---------------------------------------------------------------------------
find_package(PkgConfig REQUIRED)

pkg_check_modules(GLIB REQUIRED IMPORTED_TARGET glib-2.0>=2.32)

pkg_check_modules(GMODULE QUIET IMPORTED_TARGET gmodule-no-export-2.0>=2.32)
if(NOT GMODULE_FOUND)
    pkg_check_modules(GMODULE QUIET IMPORTED_TARGET gmodule-2.0>=2.32)
endif()
if(GMODULE_FOUND)
    set(MC_HAVE_GMODULE ON)
endif()

if(NOT MC_ENABLE_EXT2FS_ATTR OR MC_ENABLE_EXT2FS_ATTR STREQUAL "auto")
    set(_ext2fs_requested ON)
elseif(MC_ENABLE_EXT2FS_ATTR)
    set(_ext2fs_requested ON)
else()
    set(_ext2fs_requested OFF)
endif()
set(MC_HAVE_EXT2FS OFF)
if(_ext2fs_requested)
    pkg_check_modules(EXT2FS QUIET IMPORTED_TARGET "ext2fs>=1.42.4")
    pkg_check_modules(E2P QUIET IMPORTED_TARGET "e2p>=1.42.4")
    if(EXT2FS_FOUND AND E2P_FOUND)
    set(MC_HAVE_EXT2FS ON)
endif()
endif()

set(MC_HAVE_SFTP OFF)
if(MC_WITH_VFS AND MC_WITH_VFS_SFTP)
    pkg_check_modules(LIBSSH2 QUIET IMPORTED_TARGET "libssh2>=1.2.8")
    if(LIBSSH2_FOUND)
        set(MC_HAVE_SFTP ON)
    endif()
endif()

# ---------------------------------------------------------------------------
# Screen library: ncurses or slang
# ---------------------------------------------------------------------------
set(MC_USE_SLANG OFF)
set(MC_USE_NCURSES OFF)

if(NOT MC_SCREEN_LIBRARY STREQUAL "ncurses")
    pkg_check_modules(SLANG QUIET IMPORTED_TARGET "slang>=2.0")
    if(SLANG_FOUND)
        set(MC_USE_SLANG ON)
    endif()
endif()

if(MC_SCREEN_LIBRARY STREQUAL "slang")
    if(NOT MC_USE_SLANG)
        message(FATAL_ERROR "S-Lang >= 2.0.0 library not found")
    endif()
endif()

if(NOT MC_USE_SLANG)
    find_path(NCURSESW_INCLUDE_DIR ncursesw/ncurses.h)
    find_library(NCURSESW_LIBRARY NAMES ncursesw ncurses)
    if(NCURSESW_INCLUDE_DIR AND NCURSESW_LIBRARY)
        set(MC_USE_NCURSES ON)
    endif()
    find_library(TINFOW_LIBRARY NAMES tinfow tinfo)
endif()

if(NOT MC_USE_SLANG AND NOT MC_USE_NCURSES)
    message(FATAL_ERROR "Neither ncurses(w) nor S-Lang screen library was found")
endif()

set(MC_NCURSES_HAVE_WIDECHAR OFF)
set(MC_HAVE_ESCDELAY OFF)
if(MC_USE_NCURSES)
    set(CMAKE_REQUIRED_INCLUDES ${NCURSESW_INCLUDE_DIR})
    set(CMAKE_REQUIRED_LIBRARIES ${NCURSESW_LIBRARY})
    check_c_source_compiles([=[
#include <ncursesw/ncurses.h>
int main(void){ int (*p)(wint_t) = addwstr; (void)p; return 0; }
]=] _mc_addwstr)
    if(_mc_addwstr)
        set(MC_NCURSES_HAVE_WIDECHAR ON)
    endif()
    check_c_source_compiles("extern int ESCDELAY;int main(void){ESCDELAY=0;return 0;}" _mc_escdelay)
    if(_mc_escdelay)
        set(MC_HAVE_ESCDELAY ON)
    endif()
    unset(CMAKE_REQUIRED_INCLUDES)
    unset(CMAKE_REQUIRED_LIBRARIES)
endif()

# ---------------------------------------------------------------------------
# X11 support (for reading keyboard modifiers)
# ---------------------------------------------------------------------------
find_package(X11 QUIET)
set(MC_HAVE_TEXTMODE_X11_SUPPORT OFF)
set(MC_X11_LIBRARIES)
if(X11_FOUND)
    set(MC_HAVE_TEXTMODE_X11_SUPPORT ON)
    set(MC_X11_LIBRARIES ${X11_LIBRARIES})
endif()

# ---------------------------------------------------------------------------
# System probes -> config.h
# ---------------------------------------------------------------------------
set(CMAKE_REQUIRED_DEFINITIONS "${CMAKE_REQUIRED_DEFINITIONS} -D_GNU_SOURCE")
message(STATUS "DEBUG probe: CMAKE_REQUIRED_INCLUDES=${CMAKE_REQUIRED_INCLUDES} CMAKE_REQUIRED_DEFINITIONS=${CMAKE_REQUIRED_DEFINITIONS}")

# Headers (subset meaningful on the configured platform)
foreach(_hdr
    string.h memory.h limits.h malloc.h utime.h
    sys/statfs.h sys/vfs.h sys/select.h sys/ioctl.h stropts.h arpa/inet.h
    sys/socket.h pty.h libutil.h util.h gnu/libc-version.h fcntl.h os.h
    sys/param.h sys/mount.h sys/mntent.h sys/statvfs.h sys/fs/s5param.h
    sys/fs/types.h sys/ucred.h linux/fs.h)
    string(TOUPPER "${_hdr}" _hdr_up)
    string(REGEX REPLACE "[^A-Za-z0-9]" "_" _hdr_var "HAVE_${_hdr_up}")
    check_include_file("${_hdr}" _mc_hdr_ok)
    if(_mc_hdr_ok)
    set(${_hdr_var} 1)
endif()
endforeach()

# os.h does not exist on Linux/glibc; the foreach check above is spurious.
set(HAVE_OS_H 0)
# sys/fs/s5param.h is not present on modern glibc.
set(HAVE_SYS_FS_S5PARAM_H 0)
# sys/mntent.h is not present on modern glibc (use /usr/include/mntent.h).
set(HAVE_SYS_MNTENT_H 0)
# stropts.h is System V only, not on Linux/glibc.
set(HAVE_STROPTS_H 0)

set(HAVE_STRUCT_DIRENT_D_TYPE 0)
check_c_source_compiles([=[
#include <sys/types.h>
#include <dirent.h>
int main(void){struct dirent d; return d.d_type==0?0:0;}
]=] _mc_dirent_d_type)
if(_mc_dirent_d_type)
    set(HAVE_STRUCT_DIRENT_D_TYPE 1)
endif()

set(HAVE_STRUCT_STAT_ST_BLKSIZE 0)
set(HAVE_STRUCT_STAT_ST_RDEV 0)
set(HAVE_STRUCT_STAT_ST_MTIM 0)
set(HAVE_STRUCT_STAT_ST_BLOCKS 0)
check_struct_has_member("struct stat" st_blksize sys/stat.h _mc_st_blksize)
check_struct_has_member("struct stat" st_rdev sys/stat.h _mc_st_rdev)
check_struct_has_member("struct stat" st_mtim sys/stat.h _mc_st_mtim)
check_struct_has_member("struct stat" st_blocks sys/stat.h _mc_st_blocks)
if(_mc_st_blksize)
    set(HAVE_STRUCT_STAT_ST_BLKSIZE 1)
endif()
if(_mc_st_rdev)
    set(HAVE_STRUCT_STAT_ST_RDEV 1)
endif()
if(_mc_st_mtim)
    set(HAVE_STRUCT_STAT_ST_MTIM 1)
endif()
if(_mc_st_blocks)
    set(HAVE_STRUCT_STAT_ST_BLOCKS 1)
endif()

set(HAVE_STRUCT_LINGER_L_LINGER 0)
check_struct_has_member("struct linger" l_linger "sys/socket.h" _mc_l_linger
    LANGUAGE C)
if(_mc_l_linger)
    set(HAVE_STRUCT_LINGER_L_LINGER 1)
endif()

set(HAVE_STRUCT_FSSTAT_F_FSTYPENAME 0)
set(HAVE_STRUCT_STATFS_F_FSTYPENAME 0)
set(HAVE_STRUCT_STATVFS_F_BASETYPE 0)
set(HAVE_INFOMOUNT 0)
set(HAVE_INFOMOUNT_LIST 0)
set(HAVE_INFOMOUNT_QNX 0)
set(HAVE_S_SIMPLE_MSGS 0)
foreach(_n stat.statfs sys/statfs.h statfs sys/vfs.h statvfs sys/statvfs.h)
endforeach()

check_c_source_compiles([=[
#include <sys/vfs.h>
int main(void){struct statfs s; (void)s.f_fstypename; return 0;}
]=] _mc_fstypename_statfs)
if(_mc_fstypename_statfs)
    set(HAVE_STRUCT_STATFS_F_FSTYPENAME 1)
endif()

check_c_source_compiles([=[
#include <sys/stat.h>
#ifdef HAVE_SYS_STATVFS_H
#include <sys/statvfs.h>
#endif
int main(void){struct statvfs s; (void)s.f_basetype; return 0;}
]=] _mc_basetype_statvfs)
if(_mc_basetype_statvfs)
    set(HAVE_STRUCT_STATVFS_F_BASETYPE 1)
endif()

# Largefile support on 32-bit systems is granted automatically on 64-bit
# targets; expose the off_t size for the SIZEOF_OFF_T macro.
set(SIZEOF_OFF_T 0)
set(SIZEOF_LONG 0)
check_type_size("off_t" MC_SIZEOF_OFF_T)
check_type_size("long" MC_SIZEOF_LONG)
if(MC_SIZEOF_OFF_T)
    set(SIZEOF_OFF_T ${MC_SIZEOF_OFF_T})
endif()
if(MC_SIZEOF_LONG)
    set(SIZEOF_LONG ${MC_SIZEOF_LONG})
endif()

set(HAVE_S_I_SOCK ${HAVE_SYS_SOCKET_H_H})
set(HAVE_S_I_ISREG 1)
set(HAVE_S_I_DIR 1)
set(HAVE_S_I_CHR 1)
set(HAVE_S_I_BLK 1)
set(HAVE_S_I_FIFO 1)
set(HAVE_S_I_LNK 1)
set(HAVE_S_I_DOOR 0)

# Functions relevant on GNU/Linux/glibc.
# Use check_c_source_compiles (not check_function_exists) so that both a
# declaration AND a linkable symbol are required.
foreach(_func
     strverscmp strncasecmp realpath statlstat setlocale utimensat
     grantpt posix_openpt openpty copy_file_range)
    string(TOUPPER "${_func}" _func_upper)
    set(_func_var "HAVE_${_func_upper}")
      check_c_source_compiles("extern int ${_func}(); int main(void){ ${_func}(); return 0; }" _mc_fn_ok FAIL_REGEX "undefined reference")
     if(_mc_fn_ok)
        set(${_func_var} 1)
     endif()
endforeach()

# check_c_source_compiles/try_compile can be lenient about link failures in
# some CMake versions; verify statlstat explicitly.
file(WRITE ${CMAKE_BINARY_DIR}/_statlstat_test.c
     "extern int statlstat(); int main(void){ statlstat(); return 0; }")
try_compile(_statlstat_ok ${CMAKE_BINARY_DIR}
    ${CMAKE_BINARY_DIR}/_statlstat_test.c
    OUTPUT_VARIABLE _statlstat_out)
if(NOT _statlstat_ok OR _statlstat_out MATCHES "undefined reference")
    set(HAVE_STATLSTAT 0)
endif()

set(HAVE_GRANTPT 0)
set(HAVE_POSIX_OPENPT 0)
set(HAVE_OPENPTY 0)
check_function_exists(grantpt _mc_grantpt)
check_function_exists(posix_openpt _mc_posix_openpt)
check_function_exists(openpty _mc_openpty)
if(_mc_grantpt)
    set(HAVE_GRANTPT 1)
endif()
if(_mc_posix_openpt)
    set(HAVE_POSIX_OPENPT 1)
endif()
if(_mc_openpty)
    set(HAVE_OPENPTY 1)
endif()

set(HAVE_SIGHANDLER_T 0)
check_c_source_compiles([=[
#include <signal.h>
int main(void){sighandler_t s; (void)s; return 0;}
]=] _mc_sighandler)
if(_mc_sighandler)
    set(HAVE_SIGHANDLER_T 1)
endif()

# Detect MAJOR_IN_SYSMACRES: on glibc <sys/types.h> defines MAJOR_IN_SYSMACROS
# and <sys/sysmacros.h> provides major()/minor()/makedev().
check_c_source_compiles([=[
#include <sys/types.h>
#include <sys/sysmacros.h>
int main(void){ unsigned int m = major(0); (void)m; return 0; }
]=] _mc_major)
if(_mc_major)
    set(MAJOR_IN_SYSMACROS 1)
endif()

# AC_CHECK_TYPE([major_t], [unsigned int]) equivalent: when major_t is not
# a native type, define it as unsigned int (matches upstream autotools).
check_c_source_compiles([=[
#include <sys/types.h>
typedef major_t _mc_major_t_check;
int main(void){ return 0; }
]=] _mc_has_major_t)
if(NOT _mc_has_major_t)
    set(major_t "unsigned int")
endif()
check_c_source_compiles([=[
#include <sys/types.h>
typedef minor_t _mc_minor_t_check;
int main(void){ return 0; }
]=] _mc_has_minor_t)
if(NOT _mc_has_minor_t)
    set(minor_t "unsigned int")
endif()

# Detect PROMOTED_MODE_T: mode_t promotes to 'int' iff it's smaller than 'int'.
# Matches gnulib's gl_PROMOTED_TYPE_MODE_T.
check_c_source_compiles([=[
#include <sys/types.h>
typedef int array[2 * (sizeof(mode_t) < sizeof(int)) - 1];
int main(void){ return 0; }
]=] _mc_mode_t_promotes)
if(_mc_mode_t_promotes)
    set(PROMOTED_MODE_T "int")
else()
    set(PROMOTED_MODE_T "mode_t")
endif()

# Detect SIG_ATOMIC_VOLATILE_T (autotools AC_CHECK_TYPE + EGREP_HEADER).
# glibc signal.h defines sig_atomic_t and volatile sig_atomic_t.
set(_mc_sig_atomic_volatile "")
check_c_source_compiles([=[
#include <signal.h>
int main(void){ volatile sig_atomic_t v; (void)v; return 0; }
]=] _mc_sig_atomic_volatile_test)
if(_mc_sig_atomic_volatile_test)
    check_c_source_compiles([=[
#include <signal.h>
int main(void){ sig_atomic_t v; (void)v; return 0; }
]=] _mc_sig_atomic_nonvolatile_test)
    if(_mc_sig_atomic_nonvolatile_test)
        set(_mc_sig_atomic_volatile "volatile sig_atomic_t")
    else()
        set(_mc_sig_atomic_volatile "sig_atomic_t")
    endif()
endif()
if(_mc_sig_atomic_volatile)
    set(SIG_ATOMIC_VOLATILE_T "${_mc_sig_atomic_volatile}")
endif()

set(HAVE_SOCKLEN_T 0)
check_c_source_compiles([=[
#include <sys/types.h>
#include <sys/socket.h>
int main(void){socklen_t s; (void)s; return 0;}
]=] _mc_socklen)
if(_mc_socklen)
    set(HAVE_SOCKLEN_T 1)
endif()

set(HAVE_STDCKDINT_H 0)
check_include_file("stdckdint.h" _mc_stdckdint)
if(_mc_stdckdint)
    set(HAVE_STDCKDINT_H 1)
endif()

set(HAVE_G_UNIT_TESTS ${MC_ENABLE_TESTS})

set(HAVE_LIBGPM 0)
set(MC_GPM_LIBRARY)
if(MC_WITH_GPM AND CMAKE_SYSTEM_NAME MATCHES "Linux")
    find_library(MC_GPM_LIBRARY gpm)
    if(MC_GPM_LIBRARY)
    set(HAVE_LIBGPM 1)
endif()
endif()

set(MC_HAVE_LIBSSH2 ${MC_HAVE_SFTP})

if(MC_USE_SLANG)
    set(HAVE_SLANG 1)
endif()
if(MC_USE_NCURSES)
    set(HAVE_NCURSES 1)
    if(MC_NCURSES_HAVE_WIDECHAR)
    set(HAVE_NCURSES_WIDECHAR 1)
endif()
    if(MC_HAVE_ESCDELAY)
    set(HAVE_ESCDELAY 1)
endif()
    set(NCURSES_LIB_DISPLAYNAME "NCursesw")
endif()

if(MC_HAVE_TEXTMODE_X11_SUPPORT)
    set(HAVE_TEXTMODE_X11_SUPPORT 1)
endif()

if(MC_HAVE_GMODULE)
    set(HAVE_GMODULE 1)
endif()

if(MC_ENABLE_ASSERT)
    set(MC_ENABLE_ASSERT ON)
else()
    set(G_DISABLE_ASSERT 1)
endif()

if(MC_ENABLE_BACKGROUND)
    set(ENABLE_BACKGROUND 1)
endif()
if(MC_WITH_INTERNAL_EDIT)
    set(USE_INTERNAL_EDIT 1)
endif()
if(MC_WITH_DIFF_VIEWER)
    set(USE_DIFF_VIEW 1)
endif()
if(MC_ENABLE_SUBSHELL)
    set(ENABLE_SUBSHELL 1)
endif()
if(MC_ENABLE_EXT2FS_ATTR AND MC_HAVE_EXT2FS)
    set(ENABLE_EXT2FS_ATTR 1)
endif()
if(MC_WITH_VFS)
    set(ENABLE_VFS 1)
endif()
if(MC_WITH_VFS AND MC_WITH_VFS_EXTFS)
    set(ENABLE_VFS_EXTFS 1)
endif()
if(MC_WITH_VFS AND MC_WITH_VFS_SFS)
    set(ENABLE_VFS_SFS 1)
endif()
if(MC_WITH_VFS AND MC_WITH_VFS_FTP)
    set(ENABLE_VFS_FTP 1)
endif()
if(MC_WITH_VFS AND MC_WITH_VFS_SHELL)
    set(ENABLE_VFS_SHELL 1)
endif()
if(MC_WITH_VFS AND MC_WITH_VFS_TAR)
    set(ENABLE_VFS_TAR 1)
endif()
if(MC_WITH_VFS AND MC_WITH_VFS_CPIO)
    set(ENABLE_VFS_CPIO 1)
endif()
if(MC_WITH_VFS AND MC_HAVE_SFTP)
    set(ENABLE_VFS_SFTP 1)
endif()
if(MC_WITH_VFS AND (MC_WITH_VFS_FTP OR MC_WITH_VFS_SHELL OR MC_HAVE_SFTP))
    set(ENABLE_VFS_NET 1)
endif()

# File cloning detection (Linux)
set(HAVE_FILE_CLONING_BY_RANGE 0)
set(HAVE_FILE_CLONING_BY_PATH 0)
set(HAVE_FICLONERANGE 0)
if(CMAKE_SYSTEM_NAME MATCHES "Linux")
    check_include_file("linux/fs.h" _mc_linux_fs_h)
    if(_mc_linux_fs_h)
        set(HAVE_SYS_STATFS_H 1)
        check_c_source_compiles([=[
#include <linux/fs.h>
#ifndef FICLONERANGE
#error no FICLONERANGE
#endif
int main(void){return 0;}
]=] _mc_ficlonerange)
        if(_mc_ficlonerange)
            set(HAVE_FICLONERANGE 1)
            set(HAVE_FILE_CLONING_BY_RANGE 1)
        endif()
    endif()
elseif(CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
    if(HAVE_COPY_FILE_RANGE)
    set(HAVE_FILE_CLONING_BY_RANGE 1)
endif()
endif()

set(USE_FILE_CMD 0)
set(FILE_B "")
set(FILE_L "")
set(FILE_S "")
find_program(MC_FILE_CMD file)
if(MC_FILE_CMD)
    execute_process(COMMAND ${MC_FILE_CMD} -z . RESULT_VARIABLE _fc_r)
    if(_fc_r EQUAL 0)
        set(USE_FILE_CMD 1)
        set(FILE_B "-b ")
        execute_process(COMMAND ${MC_FILE_CMD} -L . RESULT_VARIABLE _fc_l)
        if(_fc_l EQUAL 0)
    set(FILE_L "-L ")
endif()
        execute_process(COMMAND ${MC_FILE_CMD} -S . RESULT_VARIABLE _fc_s)
        if(_fc_s EQUAL 0)
    set(FILE_S "-S ")
endif()
    endif()
endif()

set(HAVE_LIBUTIL_H 0)
check_include_file("libutil.h" _mc_libutil)
if(_mc_libutil)
    set(HAVE_LIBUTIL_H 1)
endif()
set(HAVE_UTIL_H 0)
check_include_file("util.h" _mc_utilh)
if(_mc_utilh)
    set(HAVE_UTIL_H 1)
endif()
set(HAVE_PTY_H 0)
check_include_file("pty.h" _mc_ptyh)
if(_mc_ptyh)
    set(HAVE_PTY_H 1)
endif()

# ---------------------------------------------------------------------------
# Directories (install locations embedded at compile time)
# ---------------------------------------------------------------------------
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "/usr/local" CACHE PATH "" FORCE)
endif()
set(MC_SYSCONFDIR "${CMAKE_INSTALL_FULL_SYSCONFDIR}/${MC_PACKAGE}/")
set(MC_DATADIR "${CMAKE_INSTALL_FULL_DATADIR}/${MC_PACKAGE}/")
set(MC_LOCALEDIR "${CMAKE_INSTALL_FULL_LOCALEDIR}")
set(MC_LIBEXECDIR "${CMAKE_INSTALL_FULL_LIBEXECDIR}/${MC_PACKAGE}/")
set(MC_EXTHELPERSDIR "${MC_LIBEXECDIR}ext.d/")
set(MC_EXTFSHELPERSDIR "${MC_LIBEXECDIR}extfs.d/")
set(MC_SAVERDIR "${MC_LIBEXECDIR}")

set(PACKAGE "${MC_PACKAGE}")
set(PACKAGE_NAME "${MC_NAME}")
set(PACKAGE_VERSION "${PROJECT_VERSION}")
set(PACKAGE_STRING "${MC_NAME} ${PROJECT_VERSION}")
set(PACKAGE_BUGREPORT "https://github.com/MidnightCommander/mc/issues")
set(VERSION "${PROJECT_VERSION}")
set(PACKAGE_TARNAME "mc")

set(USE_NLS 0)
set(ENABLE_NLS 0)

if(MC_ENABLE_CONFIGURE_ARGS)
    set(ENABLE_CONFIGURE_ARGS 1)
    set(MC_CONFIGURE_ARGS "cmake ${CMAKE_COMMAND_LINE_ARGUMENTS}")
endif()

set(ENCODING_CP1251 "CP1251")
set(ENCODING_CP866 "CP866")
set(ENCODING_ISO8859 "ISO-8859")

# Tools referenced by runtime scripts / generated files
find_program(PERL perl)
if(NOT PERL)
    set(PERL "/usr/bin/env perl")
endif()
find_program(NROFF nroff)
set(MANDOC "-man")
set(MAN_FLAGS "")
if(NROFF)
    execute_process(COMMAND ${NROFF} -mandoc OUTPUT_QUIET ERROR_QUIET
        RESULT_VARIABLE _r)
    if(_r EQUAL 0)
        set(MANDOC "-mandoc")
    endif()
    execute_process(COMMAND ${NROFF} -c OUTPUT_QUIET ERROR_QUIET
        RESULT_VARIABLE _r)
    if(_r EQUAL 0)
        set(MAN_FLAGS "-c")
    endif()
endif()

# ---------------------------------------------------------------------------
# config.h
# ---------------------------------------------------------------------------
configure_file(${CMAKE_SOURCE_DIR}/cmake/config.h.in
    ${CMAKE_BINARY_DIR}/config.h @ONLY)

# config.h.in uses "#undef SIG_ATOMIC_VOLATILE_T" as a placeholder;
# substitute the actual value when it was detected.
if(SIG_ATOMIC_VOLATILE_T)
    file(READ ${CMAKE_BINARY_DIR}/config.h _ch)
    string(REPLACE "#undef SIG_ATOMIC_VOLATILE_T"
        "#define SIG_ATOMIC_VOLATILE_T ${SIG_ATOMIC_VOLATILE_T}" _ch "${_ch}")
    file(WRITE ${CMAKE_BINARY_DIR}/config.h "${_ch}")
endif()

# When major_t/minor_t are not native types, define them as unsigned int
# (matching autotools AC_CHECK_TYPE([major_t], [unsigned int])).
if(major_t)
    file(READ ${CMAKE_BINARY_DIR}/config.h _ch)
    string(REPLACE "#undef major_t"
        "#define major_t ${major_t}" _ch "${_ch}")
    file(WRITE ${CMAKE_BINARY_DIR}/config.h "${_ch}")
endif()
if(minor_t)
    file(READ ${CMAKE_BINARY_DIR}/config.h _ch)
    string(REPLACE "#undef minor_t"
        "#define minor_t ${minor_t}" _ch "${_ch}")
    file(WRITE ${CMAKE_BINARY_DIR}/config.h "${_ch}")
endif()
if(PROMOTED_MODE_T)
    file(READ ${CMAKE_BINARY_DIR}/config.h _ch)
    string(REPLACE "#undef PROMOTED_MODE_T"
        "#define PROMOTED_MODE_T ${PROMOTED_MODE_T}" _ch "${_ch}")
    file(WRITE ${CMAKE_BINARY_DIR}/config.h "${_ch}")
endif()

mark_as_advanced(
    FORCE
    NCURSESW_INCLUDE_DIR NCURSESW_LIBRARY TINFOW_LIBRARY NCURSES_INCLUDE_PATH
    MC_GPM_LIBRARY MC_FILE_CMD)

# ---------------------------------------------------------------------------
# Top-level build
# ---------------------------------------------------------------------------
add_subdirectory(lib)
add_subdirectory(src/man2hlp)
add_subdirectory(src)
add_subdirectory(misc)
add_subdirectory(contrib)
add_subdirectory(doc)
add_subdirectory(po)

if(MC_ENABLE_TESTS)
    add_subdirectory(tests)
endif()

add_library(mc-deps INTERFACE)
target_link_libraries(mc-deps INTERFACE mc-version PkgConfig::GLIB)
if(NOT MC_USE_SLANG)
    target_link_libraries(mc-deps INTERFACE
        ${NCURSESW_LIBRARY}
        ${TINFOW_LIBRARY})
    if(NCURSESW_INCLUDE_DIR)
        target_include_directories(mc-deps INTERFACE ${NCURSESW_INCLUDE_DIR})
    endif()
endif()
if(MC_GPM_LIBRARY)
    target_link_libraries(mc-deps INTERFACE ${MC_GPM_LIBRARY})
endif()
if(MC_X11_LIBRARIES)
    target_link_libraries(mc-deps INTERFACE ${MC_X11_LIBRARIES})
endif()
if(GMODULE_FOUND)
    target_link_libraries(mc-deps INTERFACE PkgConfig::GMODULE)
endif()
if(MC_HAVE_EXT2FS)
    target_link_libraries(mc-deps INTERFACE PkgConfig::E2P)
endif()
if(MC_HAVE_SFTP)
    target_link_libraries(mc-deps INTERFACE PkgConfig::LIBSSH2)
endif()

install(FILES ${CMAKE_BINARY_DIR}/mc-version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

message(STATUS "Configuration (CMake):")
message(STATUS "  Source code location: ${CMAKE_SOURCE_DIR}")
message(STATUS "  Compiler: ${CMAKE_C_COMPILER}")
message(STATUS "  Compiler flags: ${CMAKE_C_FLAGS} ${MC_COMPILE_OPTIONS}")
message(STATUS "  Assertions: ${MC_ENABLE_ASSERT}")
message(STATUS "  Internal editor: ${MC_WITH_INTERNAL_EDIT}")
message(STATUS "  Diff viewer: ${MC_WITH_DIFF_VIEWER}")
if(MC_USE_SLANG)
    set(_mc_screenlib "S-Lang")
else()
    set(_mc_screenlib "NCurses")
endif()
message(STATUS "  Screen library: ${_mc_screenlib}")
message(STATUS "  Subshell support: ${MC_ENABLE_SUBSHELL}")
message(STATUS "  VFS: ${MC_WITH_VFS}")
