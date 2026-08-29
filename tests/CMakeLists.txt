# Unit tests require the Check framework and are only wired in when
# MC_ENABLE_TESTS is on.  This mirrors the autotools HAVE_TESTS condition.
if(NOT MC_ENABLE_TESTS)
    return()
endif()

find_package(PkgConfig REQUIRED)
pkg_check_modules(CHECK REQUIRED IMPORTED_TARGET check>=0.9.10)

message(STATUS "Unit tests will build (requires the Check framework)")
