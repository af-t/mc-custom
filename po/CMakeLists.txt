set(_SRC ${CMAKE_SOURCE_DIR})

find_program(MSGFMT msgfmt)
if(NOT MSGFMT)
    message(WARNING "msgfmt not found — translations will not be built")
    return()
endif()

# Read LINGUAS, fallback to globbing *.po
set(_linguas "")
if(EXISTS "${_SRC}/po/LINGUAS")
    file(READ "${_SRC}/po/LINGUAS" _linguas_content)
    string(REGEX MATCHALL "[a-zA-Z_@]+" _linguas "${_linguas_content}")
    # Filter out comments already stripped; keep only entries that have a .po file
    set(_filtered "")
    foreach(_l ${_linguas})
        if(EXISTS "${_SRC}/po/${_l}.po")
            list(APPEND _filtered "${_l}")
        endif()
    endforeach()
    set(_linguas "${_filtered}")
endif()
if(NOT _linguas)
    file(GLOB _po_files "${_SRC}/po/*.po")
    foreach(_po ${_po_files})
        get_filename_component(_lang "${_po}" NAME_WE)
        list(APPEND _linguas "${_lang}")
    endforeach()
endif()

set(_gmo_files "")
foreach(_lang ${_linguas})
    set(_po "${_SRC}/po/${_lang}.po")
    set(_gmo "${CMAKE_CURRENT_BINARY_DIR}/${_lang}.gmo")
    add_custom_command(
        OUTPUT "${_gmo}"
        COMMAND ${MSGFMT} -c -o "${_gmo}" "${_po}"
        DEPENDS "${_po}"
        COMMENT "Compiling ${_lang}.po -> ${_lang}.gmo"
        VERBATIM)
    list(APPEND _gmo_files "${_gmo}")
    install(FILES "${_gmo}"
        DESTINATION "${CMAKE_INSTALL_LOCALEDIR}/${_lang}/LC_MESSAGES"
        RENAME "mc.mo")
endforeach()

add_custom_target(po-gmo ALL DEPENDS ${_gmo_files})

# Optional: xgettext POT generation (maintainer target, not built by default)
find_program(XGETTEXT xgettext)
if(XGETTEXT AND EXISTS "${_SRC}/po/POTFILES.in")
    add_custom_target(pot
        COMMAND ${XGETTEXT}
            --default-domain=mc
            --keyword=_ --keyword=N_ --keyword=Q_
            --add-comments=TRANSLATORS:
            --files-from=${_SRC}/po/POTFILES.in
            --directory=${_SRC}
            --output=${CMAKE_CURRENT_BINARY_DIR}/mc.pot
        WORKING_DIRECTORY ${_SRC}
        COMMENT "Extracting mc.pot"
        VERBATIM)
endif()
