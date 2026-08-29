set(_SRC ${CMAKE_SOURCE_DIR})

# Values matching autotools substitutions for man pages
set(_man_version "${PROJECT_VERSION}")
set(_sysconfdir "${CMAKE_INSTALL_FULL_SYSCONFDIR}")
set(_libexecdir "${CMAKE_INSTALL_FULL_LIBEXECDIR}")
set(_pkglibexecdir "${CMAKE_INSTALL_FULL_LIBEXECDIR}/${MC_PACKAGE}")
set(_pkgdatadir "${CMAKE_INSTALL_FULL_DATADIR}/${MC_PACKAGE}")

# Date for man page header — use file mtime via perl if available, else current date
# Upstream uses: LC_ALL=en_US.UTF-8 perl -MPOSIX -e 'print strftime("%B %Y", localtime((lstat($file))[9]))'
# We approximate with file(TIMESTAMP) or current date.
find_program(PERL_PROG perl)
set(_man_date "")
if(PERL_PROG)
    # Try to use perl for file mtime to match upstream exactly; fall back to current date
    execute_process(
        COMMAND ${PERL_PROG} -MPOSIX -e "print POSIX::strftime(\"%B %Y\", localtime((stat(\"${_SRC}/doc/man/mc.1.in\"))[9]))"
        OUTPUT_VARIABLE _man_date
        ERROR_QUIET
        OUTPUT_STRIP_TRAILING_WHITESPACE)
endif()
if(NOT _man_date)
    string(TIMESTAMP _man_date "%B %Y" UTC)
endif()

set(_man_pages mc.1 mcview.1)
if(MC_WITH_INTERNAL_EDIT)
    list(APPEND _man_pages mcedit.1)
endif()
if(MC_WITH_DIFF_VIEWER)
    list(APPEND _man_pages mcdiff.1)
endif()

set(_generated_mans "")

foreach(_man ${_man_pages})
    get_filename_component(_base "${_man}" NAME_WE)
    set(_in "${_SRC}/doc/man/${_base}.1.in")
    set(_out "${CMAKE_CURRENT_BINARY_DIR}/${_man}")

    add_custom_command(
        OUTPUT "${_out}"
        COMMAND ${CMAKE_COMMAND}
            -DINPUT=${_in}
            -DOUTPUT=${_out}
            -DMAN_VERSION=${_man_version}
            -DDATE=${_man_date}
            -DSYSCONFDIR=${_sysconfdir}
            -DLIBEXECDIR=${_libexecdir}
            -DPKGLIBEXECDIR=${_pkglibexecdir}
            -DPKGDATADIR=${_pkgdatadir}
            -P ${CMAKE_SOURCE_DIR}/cmake/generate-man.cmake
        DEPENDS "${_in}" "${CMAKE_SOURCE_DIR}/cmake/generate-man.cmake"
        COMMENT "Generating man page ${_man}"
        VERBATIM)

    list(APPEND _generated_mans "${_out}")
    install(FILES "${_out}" DESTINATION "${CMAKE_INSTALL_MANDIR}/man1")
endforeach()

add_custom_target(man-pages ALL DEPENDS ${_generated_mans})

# Translated man pages: es hu it pl ru sr — install if .in exists
set(_man_linguas es hu it pl ru sr)
foreach(_lang ${_man_linguas})
    if(EXISTS "${_SRC}/doc/man/${_lang}/mc.1.in")
        set(_in "${_SRC}/doc/man/${_lang}/mc.1.in")
        set(_out "${CMAKE_CURRENT_BINARY_DIR}/${_lang}/mc.1")
        file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/${_lang}")
        # Use same date logic but per-language file mtime when possible
        if(PERL_PROG)
            execute_process(
                COMMAND ${PERL_PROG} -MPOSIX -e "print POSIX::strftime(\"%B %Y\", localtime((stat(\"${_in}\"))[9]))"
                OUTPUT_VARIABLE _lang_date
                ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
        endif()
        if(NOT _lang_date)
            set(_lang_date "${_man_date}")
        endif()
        add_custom_command(
            OUTPUT "${_out}"
            COMMAND ${CMAKE_COMMAND}
                -DINPUT=${_in}
                -DOUTPUT=${_out}
                -DMAN_VERSION=${_man_version}
                -DDATE=${_lang_date}
                -DSYSCONFDIR=${_sysconfdir}
                -DLIBEXECDIR=${_libexecdir}
                -DPKGLIBEXECDIR=${_pkglibexecdir}
                -DPKGDATADIR=${_pkgdatadir}
                -P ${CMAKE_SOURCE_DIR}/cmake/generate-man.cmake
            DEPENDS "${_in}" "${CMAKE_SOURCE_DIR}/cmake/generate-man.cmake"
            COMMENT "Generating man page ${_lang}/mc.1"
            VERBATIM)
        add_custom_target(man-page-${_lang} ALL DEPENDS "${_out}")
        install(FILES "${_out}" DESTINATION "${CMAKE_INSTALL_MANDIR}/${_lang}/man1")
        add_dependencies(man-pages man-page-${_lang})
    endif()
endforeach()
