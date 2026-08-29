set(_SRC ${CMAKE_SOURCE_DIR})

file(GLOB _hint_files "${_SRC}/doc/hints/l10n/mc.hint.*")
if(_hint_files)
    install(FILES ${_hint_files}
        DESTINATION "${CMAKE_INSTALL_DATADIR}/${MC_PACKAGE}/hints")
endif()
