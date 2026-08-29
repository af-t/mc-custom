# Runtime data for Midnight Commander
set(_SRC ${CMAKE_SOURCE_DIR})

set(MC_DATADIR_FULL "${CMAKE_INSTALL_FULL_DATADIR}/${MC_PACKAGE}")
set(MC_SYSCONFDIR_FULL "${CMAKE_INSTALL_FULL_SYSCONFDIR}/${MC_PACKAGE}")
set(MC_LIBEXECDIR_FULL "${CMAKE_INSTALL_FULL_LIBEXECDIR}/${MC_PACKAGE}")
set(MC_EXTFSHELPERSDIR_FULL "${MC_LIBEXECDIR_FULL}/extfs.d")
set(MC_EXTHELPERSDIR_FULL "${MC_LIBEXECDIR_FULL}/ext.d")
set(EXTFSHELPERSDIR "${MC_LIBEXECDIR_FULL}/extfs.d/")

# Relative destinations for install() — must be relative to respect --prefix
set(_mc_datadir "${CMAKE_INSTALL_DATADIR}/${MC_PACKAGE}")
set(_mc_sysconfdir "${CMAKE_INSTALL_SYSCONFDIR}/${MC_PACKAGE}")
set(_mc_libexecdir "${CMAKE_INSTALL_LIBEXECDIR}/${MC_PACKAGE}")
set(_mc_extfshelpersdir "${_mc_libexecdir}/extfs.d")
set(_mc_exthelpersdir "${_mc_libexecdir}/ext.d")

# Files generated from templates with path / tool substitution
set(MC_GENERATED_FILES)

# mc.charsets (@ENCODING_*@ placeholders become literals)
configure_file(${_SRC}/misc/mc.charsets.in
    ${CMAKE_CURRENT_BINARY_DIR}/mc.charsets @ONLY)

# mc.menu, mcedit.menu
foreach(_f mc.menu mcedit.menu)
    configure_file(${_SRC}/misc/${_f}.in
        ${CMAKE_CURRENT_BINARY_DIR}/${_f} @ONLY)
endforeach()

# ext.d helpers generated from templates
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/ext.d)
foreach(_f misc.sh text.sh)
    configure_file(${_SRC}/misc/ext.d/${_f}.in
        ${CMAKE_CURRENT_BINARY_DIR}/ext.d/${_f} @ONLY)
endforeach()

# pkgdata: mc.lib, mc.charsets -> share/mc
install(FILES ${_SRC}/misc/mc.lib
    DESTINATION ${_mc_datadir})
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/mc.charsets
    DESTINATION ${_mc_datadir})

# cfg: filehighlight, keymaps, menus -> etc/mc
install(FILES
    ${_SRC}/misc/filehighlight.ini
    ${_SRC}/misc/mc.default.keymap
    ${_SRC}/misc/mc.emacs.keymap
    ${_SRC}/misc/mc.vim.keymap
    ${CMAKE_CURRENT_BINARY_DIR}/mc.menu
    ${CMAKE_CURRENT_BINARY_DIR}/mcedit.menu
    DESTINATION ${_mc_sysconfdir})
# mc.ext.ini from .in template -> etc/mc/mc.ext.ini
install(FILES ${_SRC}/misc/mc.ext.ini.in
    DESTINATION ${_mc_sysconfdir}
    RENAME mc.ext.ini)
# edit.indent.rc is cfg_SCRIPTS (executable) -> etc/mc
install(PROGRAMS ${_SRC}/misc/edit.indent.rc
    DESTINATION ${_mc_sysconfdir})
# mc.keymap symlink -> mc.default.keymap in etc/mc (original install-data-hook)
install(CODE "
    execute_process(COMMAND \"\${CMAKE_COMMAND}\" -E create_symlink
        mc.default.keymap \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_SYSCONFDIR}/${MC_PACKAGE}/mc.keymap\")
")

# After the current script is run, the runtime "mc" install is handled
# by the C targets. Install static scripts.
install(FILES ${_SRC}/misc/ext.d/archive.sh
    ${_SRC}/misc/ext.d/doc.sh
    ${_SRC}/misc/ext.d/image.sh
    ${_SRC}/misc/ext.d/package.sh
    ${_SRC}/misc/ext.d/sound.sh
    ${_SRC}/misc/ext.d/video.sh
    ${_SRC}/misc/ext.d/web.sh
    DESTINATION ${_mc_exthelpersdir})
install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/ext.d/misc.sh
    ${CMAKE_CURRENT_BINARY_DIR}/ext.d/text.sh
    DESTINATION ${_mc_exthelpersdir})

# Skins
install(DIRECTORY ${_SRC}/misc/skins/
    DESTINATION ${_mc_datadir}/skins
    FILES_MATCHING PATTERN "*.ini" PATTERN "README.txt")

# syntax highlighting
file(READ ${_SRC}/misc/syntax/Syntax.in _syntax_in)
string(REPLACE "\${prefix}" "${CMAKE_INSTALL_PREFIX}" _syntax_out "${_syntax_in}")
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/Syntax "${_syntax_out}")
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/Syntax
    DESTINATION ${_mc_datadir}/syntax)
install(DIRECTORY ${_SRC}/misc/syntax/
    DESTINATION ${_mc_datadir}/syntax
    FILES_MATCHING PATTERN "*.syntax")

# macros.d -> share/mc/examples/macros.d (original macrodir = $(pkgdatadir)/examples/macros.d)
install(FILES ${_SRC}/misc/macros.d/macro.0.sh
    ${_SRC}/misc/macros.d/macro.1.sh
    ${_SRC}/misc/macros.d/macro.2.sh
    ${_SRC}/misc/macros.d/macro.3.sh
    ${_SRC}/misc/macros.d/macro.4.sh
    ${_SRC}/misc/macros.d/macro.5.sh
    ${_SRC}/misc/macros.d/macro.6.sh
    ${_SRC}/misc/macros.d/macro.7.sh
    DESTINATION ${_mc_datadir}/examples/macros.d)
