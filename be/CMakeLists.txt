# Copyright 2012 Cloudera Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.


cmake_minimum_required(VERSION 2.6)

# generate CTest input files
enable_testing()

# Setting this enables compiling for assembly output.  To compile to assembly:
#  1. cd into the directory containing the source file
#  2. 'make help' will list the assembly file targets (i.e. <srcfile.s>
#  3. 'make <srcfile>.s' to build the assembly for that file.  The file is built
#      to CMakeFiles/<currentdir>.dir/<srcfile>.s
PROJECT(ASSEMBLER)

# compiler flags that are common across debug/release builds
#  - msse4.2: Enable sse4.2 compiler intrinsics.
#  - Wall: Enable all warnings.
#  - Wno-sign-compare: suppress warnings for comparison between signed and unsigned 
#    integers
#  - Wunknown-pragmas: suppress warnings for unknown (compiler specific) pragmas
#  - pthread: enable multithreaded malloc
#  - DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG: enable nanosecond precision for boost
SET(CXX_COMMON_FLAGS "-msse4.2 -Wall -Wno-sign-compare -Wno-unknown-pragmas -pthread")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG")
SET(CXX_COVERAGE_FLAGS "-fprofile-arcs -ftest-coverage")

# compiler flags for different build types (run 'cmake -DCMAKE_BUILD_TYPE=<type> .')
# For all builds:
# For CMAKE_BUILD_TYPE=Debug
#   -ggdb: Enable gdb debugging
# For CMAKE_BUILD_TYPE=Release
#   -O3: Enable all compiler optimizations
#   -g: Enable symbols for profiler tools (TODO: remove for shipping)
#   -DNDEBUG: Turn off dchecks/asserts/debug only code.
#   -Wno-strict-aliasing: Suppress warnings for potential issues with pointer aliasing.  
#       Code generated by -O3 causes these warnings.
# For profile guided optimization (PGO) builds, in addition to the flags for release 
# builds:
#   1. Build first with CMAKE_BUILD_TYPE_PROFILE_GEN:
#     -fprofile-generate: Indicates compiler should insert profile guided optimization 
#      events
#   2. Run the benchmarks (generates *.gcda profiling data).
#   3. Build again with CMAKE_BUILD_TYPE_PROFILE_BUILD
#     -fprofile-use: Compiler will use the profile outputs for optimizations
SET(CXX_FLAGS_DEBUG "-g -ggdb")
SET(CXX_FLAGS_RELEASE "-O3 -g -DNDEBUG -Wno-strict-aliasing")
SET(CXX_FLAGS_PROFILE_GEN "${CXX_FLAGS_RELEASE} -fprofile-generate")
SET(CXX_FLAGS_PROFILE_BUILD "${CXX_FLAGS_RELEASE} -fprofile-use")
SET(CXX_FLAGS_ADDRESS_SANITIZER 
    "-O1 -g -fsanitize=address -fno-omit-frame-pointer -DADDRESS_SANITIZER")

# if no build build type is specified, default to debug builds
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug)
endif(NOT CMAKE_BUILD_TYPE)

STRING (TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE)

# Set compile flags based on the build type.
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG")
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_DEBUG})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE")
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_RELEASE})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "PROFILE_GEN")
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_PROFILE_GEN})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "PROFILE_BUILD")
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_PROFILE_BUILD})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "CODE_COVERAGE_DEBUG")
  SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_DEBUG} ${CXX_COVERAGE_FLAGS}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "CODE_COVERAGE_RELEASE")
  SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_RELEASE} ${CXX_COVERAGE_FLAGS}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "ADDRESS_SANITIZER")
  SET(CMAKE_CXX_COMPILER ${LLVM_CLANG_EXECUTABLE})
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_ADDRESS_SANITIZER})
else()
  message(FATAL_ERROR "Unknown build type: ${CMAKE_BUILD_TYPE}")
endif ()

# Add flags that are common across build types
#  - fverboase-asm creates better annotated assembly.  This doesn't seem to affect 
#    when building the binary.
# LLMV_CFLAGS - Adding llvm compile flags
SET(CMAKE_CXX_FLAGS "${CXX_COMMON_FLAGS} ${CMAKE_CXX_FLAGS}")
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fverbose-asm")
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${LLVM_CFLAGS}")

# Thrift requires these two definitions for some types that we use
add_definitions(-DHAVE_INTTYPES_H -DHAVE_NETINET_IN_H)

# Set clang to IR cross compile flags
# IR_COMPILE is #defined for the cross compile to remove code that bloats the IR
set(CLANG_IR_CXX_FLAGS "-emit-llvm" "-O3" "-c" "-DIR_COMPILE" "-DNDEBUG" 
  "-DHAVE_INTTYPES_H" "-DHAVE_NETINET_IN_H" "-DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG")

# setup doc generation with Doxygen
find_package(Doxygen)
if (DOXYGEN_FOUND)
  set(DOXYGEN_OUTPUT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/build/docs)
  # Possible to not input the subdirs one by one?
  set(CMAKE_DOXYGEN_INPUT
    ${CMAKE_SOURCE_DIR}/be/src
    ${CMAKE_SOURCE_DIR}/be/src/common/
    ${CMAKE_SOURCE_DIR}/be/src/exec/
    ${CMAKE_SOURCE_DIR}/be/src/exprs/
    ${CMAKE_SOURCE_DIR}/be/src/runtime/
    ${CMAKE_SOURCE_DIR}/be/src/service/
    ${CMAKE_SOURCE_DIR}/be/src/statestore/
    ${CMAKE_SOURCE_DIR}/be/src/testutil/
    ${CMAKE_SOURCE_DIR}/be/src/thrift/
    ${CMAKE_SOURCE_DIR}/be/src/util/
    ${CMAKE_SOURCE_DIR}/be/src/transport/
    )
  # CMake appends using ';'. doxygen wants spaces
  string(REPLACE ";" " " DOXYGEN_INPUT "${CMAKE_DOXYGEN_INPUT}")
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/.impala.doxy
                 ${CMAKE_CURRENT_SOURCE_DIR}/build/config/.impala.doxy)
  file(MAKE_DIRECTORY ${DOXYGEN_OUTPUT_DIR})
  add_custom_target(docs
    COMMAND ${CMAKE_COMMAND} -E echo_append "Building Docs..."
    COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/build/config/.impala.doxy
    )
else (DOXYGEN_FOUND)
  MESSAGE(STATUS "WARNING: Doxygen not found - Docs will not be created")
endif(DOXYGEN_FOUND)

# resolve "#include "<subdir>/<name>.h"
include_directories(BEFORE ${CMAKE_CURRENT_SOURCE_DIR}/src)

# resolve includes of generated code
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/generated-sources)

set(CLANG_INCLUDE_FLAGS
  "-I${CMAKE_CURRENT_SOURCE_DIR}/src"
  "-I${CMAKE_CURRENT_SOURCE_DIR}/generated-sources"
  "-I${THRIFT_INCLUDE_DIR}"
  "-I${MONGOOSE_INCLUDE_DIR}"
  "-I${GLOG_INCLUDE_DIR}"
  "-I${GFLAGS_INCLUDE_DIR}"
)

# allow linking of static libs into dynamic lib
add_definitions(-fPIC)

# set compile output directory
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG" OR
    "${CMAKE_BUILD_TYPE}" STREQUAL "CODE_COVERAGE_DEBUG" OR
    "${CMAKE_BUILD_TYPE}" STREQUAL "ADDRESS_SANITIZER")
  set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/build/debug/")
else()
  set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/build/release/")
endif()

# This is an ordered list of impala library dependencies. Individual libraries
# must not specify library dependencies in their own CMakeLists.txt file. 
# Make sure to specify link libraries in reverse order of dependency
# (ie, if libA uses symbols from libB, libA must precede libB), otherwise
# when linking statically the linker won't look for the needed symbols.
# TODO: is there a better way to specify link dependencies without having to
# list all included libs for each specific binary? any gcc flags to help with this?
# multiple alternating includes of Opcode and Exprs libs necessary to get symbols
# resolve (why?)
#
# The order of link libararies matter.
#  - Util must come before ImpalaThrift
#  - Exprs & Opcode have a circular dependency (Exprs is included twice)
set (IMPALA_LINK_LIBS
  TestUtil
  Service
  TestUtil
  CodeGen
  Service
  Exec
  Exprs
  Opcode
  Exprs
  Util
  ThriftSaslTransport
  Runtime
  Util
  Statestore
  CodeGen
  Util
  Common
  ImpalaThrift
  Service
  Exec
  Exprs
  Opcode
  Exprs
  Util
  ImpalaThrift
  Runtime
  Util
  Common
  Webserver
  GlobalFlags
# Below are all external dependencies.  They should some after the impala libs.
  ${SNAPPY_STATIC_LIB}
  ${RE2_STATIC_LIB}
  ${Boost_LIBRARIES}
  ${LLVM_MODULE_LIBS}
  thriftstatic
  thriftnbstatic
  ${SASL_LIBRARY}
  ThriftSaslTransport
  ${JAVA_JVM_LIBRARY}
  ${HDFS_LIBS}
  glogstatic
  gflagsstatic
  pprofstatic
  ${LIBEVENT}
  ${LIBZ}
  ${LIBBZ2}
  ${AVRO_STATIC_LIB}
  -lrt -lboost_date_time
)

if ("${CMAKE_BUILD_TYPE}" STREQUAL "CODE_COVERAGE")
  set (IMPALA_LINK_LIBS ${IMPALA_LINK_LIBS} -lgcov)
endif ()

# The above link list does not include tcmalloc.  This is because libfesupport
# cannot use tcmalloc in all cases.  When it is started up by the FE (for tests)
# the jvm has already made allocations before tcmalloc can be loaded.  In all
# other binaries, we can use tcmalloc except the address sanitizer build.  Address sanitizer
# is incompatible with tcmalloc (they both intercept malloc/free)
set (IMPALA_FE_SUPPORT_LINK_LIBS ${IMPALA_LINK_LIBS})
if (NOT "${CMAKE_BUILD_TYPE}" STREQUAL "ADDRESS_SANITIZER")
  set (IMPALA_LINK_LIBS ${IMPALA_LINK_LIBS} tcmallocstatic)
endif()

MESSAGE(STATUS, "Compiler Flags: ${CMAKE_CXX_FLAGS}")
MESSAGE(STATUS, "Linker Libs: ${IMPALA_LINK_LIBS}")

# Link libs for test executables.  Although not all tests need all libs,
# the build time for the tests is rather small and not worth the trouble.
set (IMPALA_TEST_LINK_LIBS ${IMPALA_LINK_LIBS} gtest)

set(LLVM_IR_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/llvm-ir")
file(MAKE_DIRECTORY ${LLVM_IR_OUTPUT_DIRECTORY})

# Utility CMake function to make specifying tests and benchmarks less verbose
FUNCTION(ADD_BE_TEST TEST_NAME)
  # This gets the directory where the test is from (e.g. 'exprs' or 'runtime')
  get_filename_component(DIR_NAME ${CMAKE_CURRENT_SOURCE_DIR} NAME)
  ADD_EXECUTABLE(${TEST_NAME} ${TEST_NAME}.cc)
  TARGET_LINK_LIBRARIES(${TEST_NAME} ${IMPALA_TEST_LINK_LIBS})
  ADD_TEST(${TEST_NAME} "${BUILD_OUTPUT_ROOT_DIRECTORY}/${DIR_NAME}/${TEST_NAME}")
ENDFUNCTION()

FUNCTION(ADD_BE_BENCHMARK BENCHMARK_NAME)
  ADD_EXECUTABLE(${BENCHMARK_NAME} ${BENCHMARK_NAME}.cc)
  TARGET_LINK_LIBRARIES(${BENCHMARK_NAME} ${IMPALA_LINK_LIBS})
ENDFUNCTION()

# compile these subdirs using their own CMakeLists.txt
add_subdirectory(src/codegen)
add_subdirectory(src/common)
add_subdirectory(src/exec)
add_subdirectory(src/exprs)
add_subdirectory(src/runtime)
add_subdirectory(src/statestore)
add_subdirectory(src/service)
add_subdirectory(src/testutil)
add_subdirectory(src/util)
add_subdirectory(src/transport)

add_subdirectory(generated-sources/opcode)

# Thrift generated files have unused variables.  Ignore those compiler
# warnings by adding this flag.  Note: impala subdirectories should be
# added *before* this so we can fix our issues.
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-variable")
add_subdirectory(generated-sources/gen-cpp)

link_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}/build/common
  ${CMAKE_CURRENT_SOURCE_DIR}/build/exec
  ${CMAKE_CURRENT_SOURCE_DIR}/build/exprs
  ${CMAKE_CURRENT_SOURCE_DIR}/build/runtime
  ${CMAKE_CURRENT_SOURCE_DIR}/build/statestore
  ${CMAKE_CURRENT_SOURCE_DIR}/build/service
  ${CMAKE_CURRENT_SOURCE_DIR}/build/testutil
  ${CMAKE_CURRENT_SOURCE_DIR}/build/thrift
  ${CMAKE_CURRENT_SOURCE_DIR}/build/util
  ${CMAKE_CURRENT_SOURCE_DIR}/build/transport
)

# only generate statically linked libs and executables
set(BUILD_SHARED_LIBS OFF)

# where to put generated libraries
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")
set(ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")

# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}")

