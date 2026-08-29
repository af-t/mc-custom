set(_SRC ${CMAKE_SOURCE_DIR})

set(LIBMC_SOURCES
    ${_SRC}/lib/terminal.c
    ${_SRC}/lib/utilunix.c
    ${_SRC}/lib/util.c
    ${_SRC}/lib/hook.c
    ${_SRC}/lib/glibcompat.c
    ${_SRC}/lib/global.c
    ${_SRC}/lib/keybind.c
    ${_SRC}/lib/lock.c
    ${_SRC}/lib/serialize.c
    ${_SRC}/lib/shell.c
    ${_SRC}/lib/timefmt.c
    ${_SRC}/lib/charsets.c
)

if(MC_MAINTAINER_MODE)
    list(APPEND LIBMC_SOURCES ${_SRC}/lib/logging.c)
endif()

add_library(libmc STATIC ${LIBMC_SOURCES})

function(mc_common_flags _tgt)
    target_include_directories(${_tgt} PRIVATE
        ${CMAKE_BINARY_DIR}
        ${_SRC})
    target_compile_options(${_tgt} PRIVATE ${MC_COMPILE_OPTIONS})
endfunction()

mc_common_flags(libmc)
target_link_libraries(libmc PUBLIC mc-deps)

function(mc_add_static_lib _name)
    set(_sources)
    foreach(_f ${ARGN})
        list(APPEND _sources ${_SRC}/${_f})
    endforeach()
    add_library(${_name} STATIC ${_sources})
    mc_common_flags(${_name})
    target_link_libraries(${_name} PUBLIC mc-deps)
endfunction()

mc_add_static_lib(mc-event-lib
    lib/event/event.c
    lib/event/manage.c
    lib/event/raise.c)

mc_add_static_lib(mc-filehighlight-lib
    lib/filehighlight/common.c
    lib/filehighlight/get-color.c
    lib/filehighlight/ini-file-read.c)

mc_add_static_lib(mc-mcconfig-lib
    lib/mcconfig/common.c
    lib/mcconfig/get.c
    lib/mcconfig/history.c
    lib/mcconfig/set.c
    lib/mcconfig/paths.c)

mc_add_static_lib(mc-search-lib
    lib/search/search.c
    lib/search/lib.c
    lib/search/normal.c
    lib/search/regex.c
    lib/search/glob.c
    lib/search/hex.c)

mc_add_static_lib(mc-skin-lib
    lib/skin/colors.c
    lib/skin/common.c
    lib/skin/hc-skins.c
    lib/skin/ini-file.c
    lib/skin/lines.c)

mc_add_static_lib(mc-strutil-lib
    lib/strutil/filevercmp.c
    lib/strutil/replace.c
    lib/strutil/strescape.c
    lib/strutil/strutil8bit.c
    lib/strutil/strutilascii.c
    lib/strutil/strutil.c
    lib/strutil/strutilutf8.c
    lib/strutil/strverscmp.c
    lib/strutil/tokenize.c
    lib/strutil/xstrtol.c)

set(_vfs_sources
    lib/vfs/direntry.c
    lib/vfs/gc.c
    lib/vfs/interface.c
    lib/vfs/parse_ls_vga.c
    lib/vfs/path.c
    lib/vfs/vfs.c
    lib/vfs/utilvfs.c)
if(ENABLE_VFS_NET)
    list(APPEND _vfs_sources lib/vfs/netutil.c)
endif()
mc_add_static_lib(mc-vfs-lib ${_vfs_sources})
target_link_libraries(mc-vfs-lib PUBLIC libmc)

set(_tty_sources
    lib/tty/color-internal.c
    lib/tty/color.c
    lib/tty/key.c
    lib/tty/keyxdef.c
    lib/tty/mouse.c
    lib/tty/tty-internal.c
    lib/tty/tty.c
    lib/tty/win.c)
if(MC_USE_SLANG)
    list(APPEND _tty_sources
        lib/tty/color-slang.c
        lib/tty/tty-slang.c)
else()
    list(APPEND _tty_sources
        lib/tty/color-ncurses.c
        lib/tty/tty-ncurses.c)
endif()
if(MC_HAVE_TEXTMODE_X11_SUPPORT)
    list(APPEND _tty_sources lib/tty/x11conn.c)
endif()
mc_add_static_lib(mc-tty-lib ${_tty_sources})

mc_add_static_lib(mc-widget-lib
    lib/widget/background.c
    lib/widget/button.c
    lib/widget/buttonbar.c
    lib/widget/check.c
    lib/widget/dialog.c
    lib/widget/dialog-switch.c
    lib/widget/frame.c
    lib/widget/gauge.c
    lib/widget/group.c
    lib/widget/groupbox.c
    lib/widget/hline.c
    lib/widget/history.c
    lib/widget/input.c
    lib/widget/input_complete.c
    lib/widget/listbox-window.c
    lib/widget/listbox.c
    lib/widget/label.c
    lib/widget/menu.c
    lib/widget/mouse.c
    lib/widget/quick.c
    lib/widget/radio.c
    lib/widget/rect.c
    lib/widget/widget-common.c
    lib/widget/wtools.c)

target_link_libraries(libmc PUBLIC
    mc-event-lib
    mc-filehighlight-lib
    mc-mcconfig-lib
    mc-search-lib
    mc-strutil-lib
    mc-skin-lib
    mc-tty-lib
    mc-vfs-lib
    mc-widget-lib)
