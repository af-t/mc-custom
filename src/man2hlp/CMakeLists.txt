set(_SRC ${CMAKE_SOURCE_DIR})

# PERL_FOR_BUILD substitution for man2hlp.in -> man2hlp
# Upstream autotools uses AC_PATH_PROG(PERL_FOR_BUILD, perl)
set(PERL_FOR_BUILD "${PERL}")

configure_file(${_SRC}/src/man2hlp/man2hlp.in
    ${CMAKE_CURRENT_BINARY_DIR}/man2hlp @ONLY)

# Make it executable in build tree (configure_file preserves permissions but
# ensure execute bit for use as tool during build)
file(COPY ${CMAKE_CURRENT_BINARY_DIR}/man2hlp
    DESTINATION ${CMAKE_CURRENT_BINARY_DIR}
    FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
                     GROUP_READ GROUP_EXECUTE
                     WORLD_READ WORLD_EXECUTE)

# man2hlp is a noinst tool — only needed at build time to generate mc.hlp.
# Expose its path to parent scope so doc/hlp can use it.
set(MAN2HLP "${CMAKE_CURRENT_BINARY_DIR}/man2hlp" PARENT_SCOPE)
set(MAN2HLP "${CMAKE_CURRENT_BINARY_DIR}/man2hlp" CACHE INTERNAL "path to man2hlp")
