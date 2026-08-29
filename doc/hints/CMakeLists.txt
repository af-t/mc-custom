set(_SRC ${CMAKE_SOURCE_DIR})

# Main hint file
install(FILES "${_SRC}/doc/hints/mc.hint"
    DESTINATION "${CMAKE_INSTALL_DATADIR}/${MC_PACKAGE}/hints")

add_subdirectory(l10n)
