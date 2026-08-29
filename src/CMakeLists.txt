set(_SRC ${CMAKE_SOURCE_DIR})

function(mc_common_flags _tgt)
    target_include_directories(${_tgt} PRIVATE
        ${CMAKE_BINARY_DIR}
        ${_SRC})
    target_compile_options(${_tgt} PRIVATE ${MC_COMPILE_OPTIONS})
endfunction()

function(mc_add_static_lib _name)
    set(_sources)
    foreach(_f ${ARGN})
        list(APPEND _sources ${_SRC}/${_f})
    endforeach()
    add_library(${_name} STATIC ${_sources})
    mc_common_flags(${_name})
    target_link_libraries(${_name} PUBLIC mc-deps)
endfunction()

# --- filemanager -----------------------------------------------------------
set(_fm_sources
    src/filemanager/achown.c
    src/filemanager/boxes.c
    src/filemanager/cd.c
    src/filemanager/chmod.c
    src/filemanager/chown.c
    src/filemanager/cmd.c
    src/filemanager/command.c
    src/filemanager/dir.c
    src/filemanager/ext.c
    src/filemanager/file.c
    src/filemanager/filegui.c
    src/filemanager/filemanager.c
    src/filemanager/find.c
    src/filemanager/hotlist.c
    src/filemanager/info.c
    src/filemanager/layout.c
    src/filemanager/mountlist.c
    src/filemanager/panelize.c
    src/filemanager/panel.c
    src/filemanager/tree.c
    src/filemanager/treestore.c)
if(ENABLE_EXT2FS_ATTR)
    list(APPEND _fm_sources src/filemanager/chattr.c)
endif()
mc_add_static_lib(mc-filemanager-lib ${_fm_sources})

# --- viewer ----------------------------------------------------------------
mc_add_static_lib(mc-viewer-lib
    src/viewer/actions_cmd.c
    src/viewer/ascii.c
    src/viewer/coord_cache.c
    src/viewer/datasource.c
    src/viewer/dialogs.c
    src/viewer/display.c
    src/viewer/growbuf.c
    src/viewer/hex.c
    src/viewer/lib.c
    src/viewer/mcviewer.c
    src/viewer/move.c
    src/viewer/nroff.c
    src/viewer/search.c)

# --- diff viewer -----------------------------------------------------------
if(MC_WITH_DIFF_VIEWER)
    mc_add_static_lib(mc-diffviewer-lib
        src/diffviewer/search.c
        src/diffviewer/ydiff.c)
endif()

# --- internal editor -----------------------------------------------------------
if(MC_WITH_INTERNAL_EDIT)
    mc_add_static_lib(mc-editor-lib
        src/editor/bookmark.c
        src/editor/edit.c
        src/editor/editcomplete.c
        src/editor/editbuffer.c
        src/editor/editcmd.c
        src/editor/editdraw.c
        src/editor/editmacros.c
        src/editor/editmenu.c
        src/editor/editoptions.c
        src/editor/editsearch.c
        src/editor/editwidget.c
        src/editor/etags.c
        src/editor/format.c
        src/editor/syntax.c)
endif()

# --- subshell -------------------------------------------------------------------
if(MC_ENABLE_SUBSHELL)
    mc_add_static_lib(mc-subshell-lib
        src/subshell/common.c
        src/subshell/proxyfunc.c)
endif()

# --- vfs tree ---------------------------------------------------------------------
set(_vfs_plugins_src src/vfs/plugins_init.c)
mc_add_static_lib(mc-vfs-plugin-lib src/vfs/plugins_init.c)

mc_add_static_lib(mc-vfs-local-lib src/vfs/local/local.c)
target_link_libraries(mc-vfs-plugin-lib PUBLIC mc-vfs-local-lib)

if(ENABLE_VFS_CPIO)
    mc_add_static_lib(mc-vfs-cpio-lib src/vfs/cpio/cpio.c)
    target_link_libraries(mc-vfs-plugin-lib PUBLIC mc-vfs-cpio-lib)
endif()
if(ENABLE_VFS_EXTFS)
    mc_add_static_lib(mc-vfs-extfs-lib src/vfs/extfs/extfs.c)
    target_compile_definitions(mc-vfs-extfs-lib PRIVATE
        LIBEXECDIR="${CMAKE_INSTALL_FULL_LIBEXECDIR}/${MC_PACKAGE}/")
    target_link_libraries(mc-vfs-plugin-lib PUBLIC mc-vfs-extfs-lib)
endif()
if(ENABLE_VFS_SHELL)
    mc_add_static_lib(mc-vfs-shell-lib
        src/vfs/shell/shell.c)
    target_compile_definitions(mc-vfs-shell-lib PRIVATE
        LIBEXECDIR="${CMAKE_INSTALL_FULL_LIBEXECDIR}/${MC_PACKAGE}/")
    target_link_libraries(mc-vfs-plugin-lib PUBLIC mc-vfs-shell-lib)
endif()
if(ENABLE_VFS_FTP)
    mc_add_static_lib(mc-vfs-ftpfs-lib
        src/vfs/ftpfs/ftpfs.c
        src/vfs/ftpfs/ftpfs_parse_ls.c)
    target_link_libraries(mc-vfs-plugin-lib PUBLIC mc-vfs-ftpfs-lib)
endif()
if(ENABLE_VFS_SFTP)
    mc_add_static_lib(mc-vfs-sftpfs-lib
        src/vfs/sftpfs/config_parser.c
        src/vfs/sftpfs/connection.c
        src/vfs/sftpfs/dir.c
        src/vfs/sftpfs/file.c
        src/vfs/sftpfs/internal.c
        src/vfs/sftpfs/sftpfs.c)
    target_link_libraries(mc-vfs-plugin-lib PUBLIC mc-vfs-sftpfs-lib)
endif()
if(ENABLE_VFS_SFS)
    mc_add_static_lib(mc-vfs-sfs-lib src/vfs/sfs/sfs.c)
    target_link_libraries(mc-vfs-plugin-lib PUBLIC mc-vfs-sfs-lib)
endif()
if(ENABLE_VFS_TAR)
    mc_add_static_lib(mc-vfs-tar-lib
        src/vfs/tar/tar-internal.c
        src/vfs/tar/tar-sparse.c
        src/vfs/tar/tar-xheader.c
        src/vfs/tar/tar.c)
    target_link_libraries(mc-vfs-plugin-lib PUBLIC mc-vfs-tar-lib)
endif()

# --- libinternal (core src code) ------------------------------------------------
set(_core_srcs
    cons.handler.c
    args.c
    clipboard.c
    events_init.c
    execute.c
    file_history.c
    help.c
    keymap.c
    learn.c
    selcodepage.c
    setup.c
    textconf.c
    usermenu.c
    util.c)
if(ENABLE_BACKGROUND)
    list(APPEND _core_srcs background.c)
endif()
set(_internal_sources)
foreach(_f ${_core_srcs})
    list(APPEND _internal_sources ${_SRC}/src/${_f})
endforeach()

add_library(libinternal STATIC ${_internal_sources})
mc_common_flags(libinternal)
target_compile_definitions(libinternal PRIVATE
    SYSCONFDIR="${MC_SYSCONFDIR}"
    LIBEXECDIR="${MC_LIBEXECDIR}"
    DATADIR="${MC_DATADIR}"
    LOCALEDIR="${MC_LOCALEDIR}"
    EXTHELPERSDIR="${MC_EXTHELPERSDIR}"
    SAVERDIR="${MC_SAVERDIR}")
target_link_libraries(libinternal PUBLIC mc-deps)
if(MC_WITH_INTERNAL_EDIT)
    # libinternal provides symbols (e.g. expand_format in usermenu.c) that
    # mc-filemanager-lib / mc-viewer-lib / mc-subshell-lib depend on, so
    # those sub-libs must link against libinternal.  We do NOT list them as
    # libinternal's own dependencies (that would reverse the link order and
    # break static linking).
endif()

# --- mc binary ---------------------------------------------------------------------
set(_mc_sources ${_SRC}/src/main.c)
add_executable(mc ${_mc_sources})
mc_common_flags(mc)

target_link_libraries(mc PRIVATE
    mc-filemanager-lib
    mc-viewer-lib
    mc-vfs-plugin-lib
    libinternal
    libmc)
if(MC_WITH_INTERNAL_EDIT)
    target_link_libraries(mc PRIVATE mc-editor-lib)
endif()
if(MC_WITH_DIFF_VIEWER)
    target_link_libraries(mc PRIVATE mc-diffviewer-lib)
endif()
if(MC_ENABLE_SUBSHELL)
    target_link_libraries(mc PRIVATE mc-subshell-lib)
endif()

target_compile_definitions(mc PRIVATE
    SYSCONFDIR="${MC_SYSCONFDIR}"
    LIBEXECDIR="${MC_LIBEXECDIR}"
    DATADIR="${MC_DATADIR}"
    LOCALEDIR="${MC_LOCALEDIR}")

if(MC_USE_SLANG)
    if(SLANG_FOUND)
        target_link_libraries(mc PRIVATE PkgConfig::SLANG)
    endif()
endif()

if(MC_MAINTAINER_MODE)
    target_compile_definitions(libinternal PRIVATE USE_MAINTAINER_MODE=1)
endif()

install(TARGETS mc DESTINATION ${CMAKE_INSTALL_BINDIR})

# symlinks mcview / mcedit / mcdiff
# Use install-time CMAKE_INSTALL_PREFIX to respect `cmake --install --prefix`
set(_mc_links mcview)
if(MC_WITH_INTERNAL_EDIT)
    list(APPEND _mc_links mcedit)
endif()
if(MC_WITH_DIFF_VIEWER)
    list(APPEND _mc_links mcdiff)
endif()
foreach(_link ${_mc_links})
    install(CODE "
        execute_process(COMMAND
            \"${CMAKE_COMMAND}\" -E create_symlink mc
            \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_BINDIR}/${_link}\")")
endforeach()

# console saver (Linux)
if(CMAKE_SYSTEM_NAME MATCHES "Linux")
    add_executable(cons.saver ${_SRC}/src/consaver/cons.saver.c)
    mc_common_flags(cons.saver)
    target_link_libraries(cons.saver PRIVATE mc-deps)
    if(MC_USE_SLANG)
        if(SLANG_FOUND)
            target_link_libraries(cons.saver PRIVATE PkgConfig::SLANG)
        endif()
    endif()
    install(TARGETS cons.saver DESTINATION ${CMAKE_INSTALL_LIBEXECDIR}/${MC_PACKAGE})
endif()

# --- sfs.ini (sysconf) --------------------------------------------------------
if(ENABLE_VFS_SFS)
    install(FILES ${_SRC}/src/vfs/sfs/sfs.ini
        DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/${MC_PACKAGE})
endif()

# --- shell helpers ------------------------------------------------------------
if(ENABLE_VFS_SHELL)
    set(_shell_helpers
        README.shell append chmod chown fexists get hardlink info ln ls
        mkdir mv rmdir send unlink utime)
    set(_shell_srcs "")
    foreach(_h ${_shell_helpers})
        list(APPEND _shell_srcs ${_SRC}/src/vfs/shell/helpers/${_h})
    endforeach()
    install(FILES ${_shell_srcs}
        DESTINATION ${CMAKE_INSTALL_LIBEXECDIR}/${MC_PACKAGE}/shell)
endif()

# --- extfs helpers ------------------------------------------------------------
if(ENABLE_VFS_EXTFS)
    file(GLOB _extfs_in "${_SRC}/src/vfs/extfs/helpers/*.in")
    set(_extfs_generated "")
    foreach(_in ${_extfs_in})
        get_filename_component(_name "${_in}" NAME_WE)
        set(_out "${CMAKE_CURRENT_BINARY_DIR}/extfs.d/${_name}")
        file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/extfs.d")
        # Substitute @PERL@ — use configure_file with @ONLY but the file uses @PERL@
        set(PERL "${PERL}")
        configure_file("${_in}" "${_out}" @ONLY)
        list(APPEND _extfs_generated "${_out}")
    endforeach()
    # Non-template helpers (already executables / scripts)
    file(GLOB _extfs_plain "${_SRC}/src/vfs/extfs/helpers/*")
    set(_extfs_plain_filtered "")
    foreach(_f ${_extfs_plain})
        if(NOT _f MATCHES "\\.in$" AND NOT _f MATCHES "/\\.gitignore$" AND NOT IS_DIRECTORY "${_f}")
            get_filename_component(_fname "${_f}" NAME)
            if(NOT _fname MATCHES "^README")
                list(APPEND _extfs_plain_filtered "${_f}")
            endif()
        endif()
    endforeach()
    # Install generated + plain helpers to libexec/mc/extfs.d
    if(_extfs_generated)
        install(PROGRAMS ${_extfs_generated}
            DESTINATION ${CMAKE_INSTALL_LIBEXECDIR}/${MC_PACKAGE}/extfs.d)
    endif()
    if(_extfs_plain_filtered)
        install(PROGRAMS ${_extfs_plain_filtered}
            DESTINATION ${CMAKE_INSTALL_LIBEXECDIR}/${MC_PACKAGE}/extfs.d)
    endif()
endif()
