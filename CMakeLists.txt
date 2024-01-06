cmake_minimum_required (VERSION 2.6)
project (assembly-stats)
include(ExternalProject)

set(CMAKE_BUILD_TYPE "Release")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wpedantic")

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

add_library(fasta ${CMAKE_CURRENT_SOURCE_DIR}/src/fasta.cpp)
add_library(fastq ${CMAKE_CURRENT_SOURCE_DIR}/src/fastq.cpp)
add_library(filetype ${CMAKE_CURRENT_SOURCE_DIR}/src/filetype.cpp)
add_library(stats ${CMAKE_CURRENT_SOURCE_DIR}/src/stats.cpp)
add_executable(assembly-stats ${CMAKE_CURRENT_SOURCE_DIR}/src/assembly-stats.cpp)

if(CMAKE_BUILD_TESTS)
  add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/external/gtest-1.7.0)
  enable_testing()
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/external/gtest-1.7.0/include)
  add_executable(runUnitTests
    ${CMAKE_CURRENT_SOURCE_DIR}/test/src/fasta_unittest.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/test/src/fastq_unittest.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/test/src/filetype_unittest.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/test/src/stats_unittest.cpp)
  target_link_libraries(runUnitTests gtest gtest_main fasta fastq filetype stats)
  add_test(runUnitTests runUnitTests)
endif()

## Check supported compression types
find_package(BZip2)
if (BZIP2_FOUND)
  include_directories(${BZIP2_INCLUDE_DIRS})
  target_link_libraries(fasta fastq filetype stats ${BZIP2_LIBRARIES})
endif()
find_package(LibLZMA)
if (LIBLZMA_FOUND)
  include_directories(${LIBLZMA_INCLUDE_DIRS})
  target_link_libraries(fasta fastq filetype stats ${LIBLZMA_LIBRARIES})
endif()
find_package(ZLIB REQUIRED)
if (ZLIB_FOUND)
  include_directories(${ZLIB_INCLUDE_DIRS})
  target_link_libraries(fasta fastq filetype stats ${ZLIB_LIBRARIES})
endif()

## Download dependencies if not provided
## bxzstr
if (DEFINED CMAKE_BXZSTR_HEADERS)
  message(STATUS "bxzstr headers provided in: ${CMAKE_BXZSTR_HEADERS}")
else()
  ExternalProject_Add(bxzstr
    GIT_REPOSITORY    https://github.com/tmaklin/bxzstr.git
    GIT_TAG           v1.2.1
    PREFIX            "external"
    SOURCE_DIR        ${CMAKE_CURRENT_SOURCE_DIR}/external/bxzstr
    CMAKE_ARGS        -D CMAKE_MODULE_PATH=${CMAKE_CURRENT_SOURCE_DIR}/external/bxzstr/cmake
    BUILD_IN_SOURCE   1
    INSTALL_COMMAND   ""
    )
  set(CMAKE_BXZSTR_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/external/bxzstr/include)
  set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/external/bxzstr/cmake)
  find_package(Zstd REQUIRED)
  target_link_libraries(assembly-stats zstd)
  if (CMAKE_BUILD_TESTS)
    target_link_libraries(runUnitTests zstd)
  endif()
  add_dependencies(fasta bxzstr)
  add_dependencies(fastq bxzstr)
  add_dependencies(filetype bxzstr)
  add_dependencies(stats bxzstr)
  add_dependencies(assembly-stats bxzstr)
endif()
include_directories(${CMAKE_BXZSTR_HEADERS})

## agc
set(CMAKE_AGC_LIBRARY "/home/temaklin/software/agc/libagc.a")
set(CMAKE_AGC_HEADERS "/home/temaklin/software/agc/src/core")
target_link_libraries(assembly-stats ${CMAKE_AGC_LIBRARY})
include_directories(${CMAKE_AGC_HEADERS})

## agc-istream
set(CMAKE_AGC-ISTREAM_HEADERS "/home/temaklin/software/agc_istream")
include_directories(${CMAKE_AGC-ISTREAM_HEADERS})

target_link_libraries(stats fasta fastq filetype)
target_link_libraries(assembly-stats stats)

set(INSTALL_DIR /usr/local/bin CACHE PATH "Installation directory for executables")
install(TARGETS assembly-stats DESTINATION ${INSTALL_DIR})
