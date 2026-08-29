add_subdirectory(hints)
add_subdirectory(man)
add_subdirectory(hlp)

# Top-level doc files — kept as EXTRA_DIST in autotools, no install needed.
# If needed, they can be installed to docdir:
# install(FILES FAQ HACKING COPYING README TODO INSTALL NEWS
#     DESTINATION ${CMAKE_INSTALL_DOCDIR})
