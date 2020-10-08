cmake_minimum_required(VERSION 3.7)

project(libworktipsmq CXX C)

include(GNUInstallDirs)

set(WORKTIPSMQ_VERSION_MAJOR 1)
set(WORKTIPSMQ_VERSION_MINOR 2)
set(WORKTIPSMQ_VERSION_PATCH 0)
set(WORKTIPSMQ_VERSION "${WORKTIPSMQ_VERSION_MAJOR}.${WORKTIPSMQ_VERSION_MINOR}.${WORKTIPSMQ_VERSION_PATCH}")
message(STATUS "worktipsmq v${WORKTIPSMQ_VERSION}")

set(WORKTIPSMQ_LIBVERSION 0)


option(BUILD_SHARED_LIBS "Build shared libraries instead of static ones" ON)


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

configure_file(worktipsmq/version.h.in worktipsmq/version.h @ONLY)
configure_file(libworktipsmq.pc.in libworktipsmq.pc @ONLY)

add_library(worktipsmq
    worktipsmq/address.cpp
    worktipsmq/auth.cpp
    worktipsmq/bt_serialize.cpp
    worktipsmq/connections.cpp
    worktipsmq/jobs.cpp
    worktipsmq/worktipsmq.cpp
    worktipsmq/proxy.cpp
    worktipsmq/worker.cpp
)
set_target_properties(worktipsmq PROPERTIES SOVERSION ${WORKTIPSMQ_LIBVERSION})

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(worktipsmq PRIVATE Threads::Threads)

# libzmq is nearly impossible to link statically from a system-installed static library: it depends
# on a ton of other libraries, some of which are not all statically available.  If the caller wants
# to mess with this, so be it: they can set up a libzmq target and we'll use it.  Otherwise if they
# asked us to do things statically, don't even try to find a system lib and just build it.
set(worktipsmq_build_static_libzmq OFF)
if(TARGET libzmq)
    target_link_libraries(worktipsmq PUBLIC libzmq)
elseif(BUILD_SHARED_LIBS)
    include(FindPkgConfig)
    pkg_check_modules(libzmq libzmq>=4.3 IMPORTED_TARGET)

    if(libzmq_FOUND)
        # Debian sid includes a -isystem in the mit-krb package that, starting with pkg-config 0.29.2,
        # breaks cmake's pkgconfig module because it stupidly thinks "-isystem" is a path, so if we find
        # -isystem in the include dirs then hack it out.
        get_property(zmq_inc TARGET PkgConfig::libzmq PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
        list(FIND zmq_inc "-isystem" broken_isystem)
        if(NOT broken_isystem EQUAL -1)
          list(REMOVE_AT zmq_inc ${broken_isystem})
          set_property(TARGET PkgConfig::libzmq PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${zmq_inc})
        endif()

        target_link_libraries(worktipsmq PUBLIC PkgConfig::libzmq)
    else()
        set(worktipsmq_build_static_libzmq ON)
    endif()
else()
    set(worktipsmq_build_static_libzmq ON)
endif()

if(worktipsmq_build_static_libzmq)
    message(STATUS "libzmq >= 4.3 not found or static build requested, building bundled version")
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/local-libzmq")
    include(LocalLibzmq)
    target_link_libraries(worktipsmq PUBLIC libzmq_vendor)
endif()

target_include_directories(worktipsmq
    PUBLIC
        $<INSTALL_INTERFACE:>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/cppzmq>
)

target_compile_options(worktipsmq PRIVATE -Wall -Wextra -Werror)
set_target_properties(worktipsmq PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
    POSITION_INDEPENDENT_CODE ON
)

function(link_dep_libs target linktype libdirs)
    foreach(lib ${ARGN})
        find_library(link_lib-${lib} NAMES ${lib} PATHS ${libdirs})
        message(STATUS "FIND ${lib} FOUND ${link_lib-${lib}}")
        if(link_lib-${lib})
            target_link_libraries(${target} ${linktype} ${link_lib-${lib}})
        endif()
    endforeach()
endfunction()

# If the caller has already set up a sodium target then we will just link to it, otherwise we go
# looking for it.
if(TARGET sodium)
    target_link_libraries(worktipsmq PUBLIC sodium)
    if(worktipsmq_build_static_libzmq)
        target_link_libraries(libzmq_vendor INTERFACE sodium)
    endif()
else()
    pkg_check_modules(sodium REQUIRED libsodium IMPORTED_TARGET)

    if(BUILD_SHARED_LIBS)
        target_link_libraries(worktipsmq PUBLIC PkgConfig::sodium)
        if(worktipsmq_build_static_libzmq)
            target_link_libraries(libzmq_vendor INTERFACE PkgConfig::sodium)
        endif()
    else()
        link_dep_libs(worktipsmq PUBLIC "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
        target_include_directories(worktipsmq PUBLIC ${sodium_STATIC_INCLUDE_DIRS})
        if(worktipsmq_build_static_libzmq)
            link_dep_libs(libzmq_vendor INTERFACE "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
            target_link_libraries(libzmq_vendor INTERFACE ${sodium_STATIC_INCLUDE_DIRS})
        endif()
    endif()
endif()

add_library(worktipsmq::worktipsmq ALIAS worktipsmq)

export(
    TARGETS worktipsmq
    NAMESPACE worktipsmq::
    FILE worktipsmqTargets.cmake
)
install(
    TARGETS worktipsmq
    EXPORT worktipsmqConfig
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(
    FILES worktipsmq/address.h
          worktipsmq/auth.h
          worktipsmq/base32z.h
          worktipsmq/base64.h
          worktipsmq/batch.h
          worktipsmq/bt_serialize.h
          worktipsmq/bt_value.h
          worktipsmq/connections.h
          worktipsmq/hex.h
          worktipsmq/worktipsmq.h
          worktipsmq/message.h
          worktipsmq/string_view.h
          ${CMAKE_CURRENT_BINARY_DIR}/worktipsmq/version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/worktipsmq
)

option(WORKTIPSMQ_INSTALL_CPPZMQ "Install cppzmq header with worktipsmq/ headers" ON)
if(WORKTIPSMQ_INSTALL_CPPZMQ)
  install(
    FILES cppzmq/zmq.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/worktipsmq
  )
endif()


install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/libworktipsmq.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
  set(worktipsmq_IS_TOPLEVEL_PROJECT TRUE)
else()
  set(worktipsmq_IS_TOPLEVEL_PROJECT FALSE)
endif()

option(WORKTIPSMQ_BUILD_TESTS "Building and perform worktipsmq tests" ${worktipsmq_IS_TOPLEVEL_PROJECT})
if(WORKTIPSMQ_BUILD_TESTS)
    add_subdirectory(tests)
endif()

