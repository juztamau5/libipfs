project(libipfs)

cmake_minimum_required(VERSION 2.8)

enable_language(CXX)

set(libipfs_NAME libipfs)
set(libipfs_DESCRIPTION "IPFS library")
set(libipfs_VERSION_MAJOR 0)
set(libipfs_VERSION_MINOR 0)
set(libipfs_VERSION_PATCH 1)

################################################################################
#
#  CMake modules
#
################################################################################

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

find_package(Go 1.5)

include(ExternalProject)
include(cmake/UseMultiArch.cmake)

################################################################################
#
#  Configure Go and build IPSF
#
################################################################################

if (GO_FOUND)
  # IPSF repo, without the leading protocol
  set(GO_IPSF_REPO github.com/juztamau5/go-ipfs)

  # For more information, run `go help environment`
  set(ENV{GOPATH} "${CMAKE_BINARY_DIR}")

  # Available OSes: linux, darwin, windows, android, netbsd, ...?
  if(WIN32)
    set(ENV{GOOS} windows)
  elseif(APPLE)
    set(ENV{GOOS} darwin)
  elseif(UNIX)
    set(ENV{GOOS} linux) # TODO
  else()
    message(FATAL_ERROR "Unknown system for Go compiler")
  endif()

  # Available architectures: amd64, 386, arm, arm64, ppc64, ppc64le, ...?
  set(ENV{GOARCH} amd64) # TODO

  # cgo is disabled by default when cross-compiling (https://golang.org/cmd/cgo)
  set(ENV{CGO_ENABLED} 1)

  message(STATUS "Fetching and building IPFS for $ENV{GOOS}_$ENV{GOARCH}")

  execute_process(COMMAND ${GO_EXECUTABLE} get -u -v -buildmode=c-archive ${GO_IPSF_REPO}/cmd/ipfs)

  set(GO_IPSF_OUTPUT_DIR "$ENV{GOPATH}/pkg/$ENV{GOOS}_$ENV{GOARCH}/${GO_IPSF_REPO}/cmd")

  set(GO_IPSF_LIBRARY "${GO_IPSF_OUTPUT_DIR}/ipfs${CMAKE_STATIC_LIBRARY_SUFFIX}")

  if (EXISTS ${GO_IPSF_LIBRARY})
    set(GO_IPFS_FOUND true)
    include_directories(${GO_IPSF_OUTPUT_DIR})
    add_definitions(-DHAVE_GO_IPSF)
  else()
    message(WARNING "Can't find IPFS library: ${GO_IPSF_LIBRARY}")
  endif()
else()
  # Change warning to status when go-ipfs no longer required
  message(WARNING "Go not found. Continuing without go-ipfs")
endif()

################################################################################
#
#  Add sources, headers and libraries
#
################################################################################

include_directories(${PROJECT_SOURCE_DIR}/include)

set(IPSF_SOURCES)

set(STANDALONE_SOURCES
    src/main.cpp)

set(LIBRARY_SOURCES
    src/lib.cpp)

if (GO_IPFS_FOUND)
  list(APPEND DEPENDENCIES ${GO_IPSF_LIBRARY}
                           pthread)
endif()

install(FILES include/ipfs.h
              include/ipfs/libipfs.h
        DESTINATION include/ipfs)

################################################################################
#
#  Standalone target
#
################################################################################

### HACK: don't build standalone target on OSX
if(NOT APPLE)
  add_executable(ipfs_ctrl ${IPSF_SOURCES} ${STANDALONE_SOURCES})

  target_link_libraries(ipfs_ctrl ${DEPENDENCIES})
endif()

################################################################################
#
#  Library target
#
################################################################################

add_library(ipfs STATIC ${IPSF_SOURCES} ${LIBRARY_SOURCES})

install(TARGETS ipfs ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${GO_IPSF_LIBRARY}
        DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RENAME libgo-ipfs${CMAKE_STATIC_LIBRARY_SUFFIX})

configure_file(cmake/libipfs-config.cmake.in
               libipfs-config.cmake @ONLY)
install(FILES ${CMAKE_BINARY_DIR}/libipfs-config.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR_NOARCH}/libipfs)

################################################################################
#
#  Warnings
#
################################################################################

if(MSVC)
    # Force to always compile with W4
    if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
        string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
    endif()
elseif(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
    # Update if necessary
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-long-long -Wpedantic")
endif()
