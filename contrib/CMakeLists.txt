set(_SRC ${CMAKE_SOURCE_DIR})

# Variables for configure_file() to substitute @pkglibexecdir@ and @bindir@
# Must match the values used by autotools pkglibexecdir = $(libexecdir)/@PACKAGE@
set(pkglibexecdir "${CMAKE_INSTALL_FULL_LIBEXECDIR}/${MC_PACKAGE}")
set(bindir "${CMAKE_INSTALL_FULL_BINDIR}")

configure_file(${_SRC}/contrib/mc.sh.in
    ${CMAKE_CURRENT_BINARY_DIR}/mc.sh @ONLY)
configure_file(${_SRC}/contrib/mc.fish.in
    ${CMAKE_CURRENT_BINARY_DIR}/mc.fish @ONLY)
configure_file(${_SRC}/contrib/mc.csh.in
    ${CMAKE_CURRENT_BINARY_DIR}/mc.csh @ONLY)
configure_file(${_SRC}/contrib/mc-wrapper.sh.in
    ${CMAKE_CURRENT_BINARY_DIR}/mc-wrapper.sh @ONLY)
configure_file(${_SRC}/contrib/mc-wrapper.fish.in
    ${CMAKE_CURRENT_BINARY_DIR}/mc-wrapper.fish @ONLY)
configure_file(${_SRC}/contrib/mc-wrapper.csh.in
    ${CMAKE_CURRENT_BINARY_DIR}/mc-wrapper.csh @ONLY)

# pkglibexec_SCRIPTS equivalent — install to libexec/mc
install(PROGRAMS
    ${CMAKE_CURRENT_BINARY_DIR}/mc.sh
    ${CMAKE_CURRENT_BINARY_DIR}/mc.fish
    ${CMAKE_CURRENT_BINARY_DIR}/mc.csh
    ${CMAKE_CURRENT_BINARY_DIR}/mc-wrapper.sh
    ${CMAKE_CURRENT_BINARY_DIR}/mc-wrapper.fish
    ${CMAKE_CURRENT_BINARY_DIR}/mc-wrapper.csh
    DESTINATION "${CMAKE_INSTALL_LIBEXECDIR}/${MC_PACKAGE}")