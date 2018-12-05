#            Copyright (c) Dean Michael Berris 2010.
# Distributed under the Boost Software License, Version 1.0.
#    (See accompanying file LICENSE_1_0.txt or copy at
#          http://www.boost.org/LICENSE_1_0.txt)

cmake_minimum_required(VERSION 2.8)
project(CPP-NETLIB)

option( CPP-NETLIB_BUILD_SHARED_LIBS "Build cpp-netlib as shared libraries." OFF )
option( CPP-NETLIB_BUILD_TESTS "Build the cpp-netlib project tests." ON)
option( CPP-NETLIB_BUILD_EXAMPLES "Build the cpp-netlib project examples." ON)

include(GNUInstallDirs)

# determine install path for CMake config files
if(WIN32 AND NOT CYGWIN)
    set(DEF_INSTALL_CMAKE_DIR CMake)
else()
    set(DEF_INSTALL_CMAKE_DIR lib/cmake/cppnetlib)
endif()
set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH "Installation directory for CMake files")

# Make relative cmake install path absolute (needed later on)
if(NOT IS_ABSOLUTE "${INSTALL_CMAKE_DIR}")
    set(INSTALL_CMAKE_DIR "${CMAKE_INSTALL_PREFIX}/${INSTALL_CMAKE_DIR}")
endif()


if(CPP-NETLIB_BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
  message (STATUS "Linking boost testing libs dynamically...")
  set(Boost_USE_STATIC_LIBS OFF)
  set(CPP-NETLIB_BUILD_SHARED_LIBS ON)
  set(BUILD_SHARED_LIBS ON)
  add_definitions(-DBOOST_TEST_DYN_LINK)
else()
  set(Boost_USE_STATIC_LIBS ON)
  set(CPP-NETLIB_BUILD_SHARED_LIBS OFF)
  set(BUILD_SHARED_LIBS OFF)
endif()

set(Boost_USE_MULTI_THREADED ON)
find_package( Boost 1.54.0
    REQUIRED unit_test_framework system regex date_time thread filesystem
    program_options chrono atomic )
find_package( OpenSSL )
find_package( Threads )
set(CMAKE_VERBOSE_MAKEFILE true)

set(CPPNETLIB_VERSION_MAJOR 0) # MUST bump this whenever we make ABI-incompatible changes
set(CPPNETLIB_VERSION_MINOR 11)
set(CPPNETLIB_VERSION_PATCH 0)
set(CPPNETLIB_VERSION_STRING ${CPPNETLIB_VERSION_MAJOR}.${CPPNETLIB_VERSION_MINOR}.${CPPNETLIB_VERSION_PATCH})

if (CMAKE_BUILD_TYPE MATCHES Debug)
    add_definitions(-DBOOST_NETWORK_DEBUG)
endif()

if (OPENSSL_FOUND)
    add_definitions(-DBOOST_NETWORK_ENABLE_HTTPS)
    include_directories(${OPENSSL_INCLUDE_DIR})
endif()

if (${CMAKE_CXX_COMPILER_ID} MATCHES GNU)
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
elseif (${CMAKE_CXX_COMPILER_ID} MATCHES Clang)
  if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    # We want to link in C++11 mode if we're using Clang and on OS X.
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -ftemplate-depth=256 -std=c++11 -stdlib=libc++")
  else()
    # We just add the -Wall and a high enough template depth
    # flag for Clang in other systems.
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -ftemplate-depth=256")
  endif()
endif()


if (Boost_FOUND)
    if (MSVC)
      add_definitions(-D_SCL_SECURE_NO_WARNINGS)
    endif(MSVC)
    if (WIN32)
      add_definitions(-D_WIN32_WINNT=0x0501)
    endif(WIN32)
    include_directories(${Boost_INCLUDE_DIRS})
    enable_testing()
    add_subdirectory(libs/network/src)
    if (CPP-NETLIB_BUILD_TESTS)
      add_subdirectory(libs/network/test)
    endif (CPP-NETLIB_BUILD_TESTS)
    add_subdirectory(libs/network/experiment)
    if (NOT MSVC AND CPP-NETLIB_BUILD_TESTS)
      add_subdirectory(libs/mime/test)
    endif(NOT MSVC AND CPP-NETLIB_BUILD_TESTS)
    if (CPP-NETLIB_BUILD_EXAMPLES)
      add_subdirectory(libs/network/example)
    endif (CPP-NETLIB_BUILD_EXAMPLES)
endif(Boost_FOUND)

if (MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj")
endif()

enable_testing()

install(DIRECTORY boost DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

###
## Export Targets
# (so cpp-netlib can be easily used by other CMake projects)
# [see http://www.cmake.org/Wiki/CMake/Tutorials/How_to_create_a_ProjectConfig.cmake_file]

# Add all targets to the build-tree export set
export(TARGETS cppnetlib-client-connections cppnetlib-server-parsers cppnetlib-uri
    FILE "${PROJECT_BINARY_DIR}/cppnetlibTargets.cmake")
# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
export(PACKAGE cppnetlib)
# Create the cppnetlibConfig.cmake and cppnetlibConfigVersion files
file(RELATIVE_PATH REL_INCLUDE_DIR "${INSTALL_CMAKE_DIR}"
    "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
# ... for the build tree
set(CONF_INCLUDE_DIRS "${PROJECT_SOURCE_DIR}")
configure_file(cppnetlibConfig.cmake.in
    "${PROJECT_BINARY_DIR}/cppnetlibConfig.cmake" @ONLY)
# ... for the install tree
set(CONF_INCLUDE_DIRS "\${CPPNETLIB_CMAKE_DIR}/${REL_INCLUDE_DIR}")
configure_file(cppnetlibConfig.cmake.in
    "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/cppnetlibConfig.cmake" @ONLY)
# ... for both
configure_file(cppnetlibConfigVersion.cmake.in
    "${PROJECT_BINARY_DIR}/cppnetlibConfigVersion.cmake" @ONLY)
# Install the cppnetlibConfig.cmake and cppnetlibConfigVersion.cmake
install(FILES
    "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/cppnetlibConfig.cmake"
    "${PROJECT_BINARY_DIR}/cppnetlibConfigVersion.cmake"
    DESTINATION "${INSTALL_CMAKE_DIR}"
    COMPONENT dev)
# Install the export set for use with the install-tree
install(EXPORT cppnetlibTargets
    DESTINATION "${INSTALL_CMAKE_DIR}"
    COMPONENT dev)
