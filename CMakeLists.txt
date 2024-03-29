if("x${CMAKE_SOURCE_DIR}" STREQUAL "x${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR "\
In-source build is not a good practice.
Please use:
  mkdir build
  cd build
  cmake ..
to build this project"
  )
endif()

cmake_minimum_required(VERSION 3.8 FATAL_ERROR)

project(k2 CUDA CXX)

set(K2_VERSION "0.3.3")

# ----------------- Supported build types for K2 project -----------------
set(ALLOWABLE_BUILD_TYPES Debug Release RelWithDebInfo MinSizeRel)
set(DEFAULT_BUILD_TYPE "Debug")
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "${ALLOWABLE_BUILD_TYPES}")
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  # CMAKE_CONFIGURATION_TYPES: with config type values from other generators (IDE).
  message(STATUS "No CMAKE_BUILD_TYPE given, default to Debug")
  set(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}")
elseif(NOT CMAKE_BUILD_TYPE IN_LIST ALLOWABLE_BUILD_TYPES)
  message(FATAL_ERROR "Invalid build type: ${CMAKE_BUILD_TYPE}, \
    choose one from ${ALLOWABLE_BUILD_TYPES}")
endif()

# About c++ std: pybind11, cub, conan require 11, and cub prefers 14.
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_EXTENSIONS OFF)
message(STATUS "C++ Standard version: ${CMAKE_CXX_STANDARD}")

# init CMAKE_MODULE_PATH as build-dir, where conan generates *.cmake
list(PREPEND CMAKE_MODULE_PATH CMAKE_BINARY_DIR "${CMAKE_SOURCE_DIR}/build")

if("${CMAKE_BUILD_TYPE_UPPERCASE}" STREQUAL "DEBUG")
  # refer to https://docs.nvidia.com/cuda/cuda-memcheck/index.html#compilation-options
  # The two options are to make cuda-memcheck's stack backtrace feature more useful.
  set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --compiler-options -rdynamic --compiler-options -lineinfo")
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(BUILD_SHARED_LIBS "Whether to build shared or static lib" ON)
option(K2_USE_PYTORCH "Whether to build with PyTorch" ON)
option(K2_ENABLE_NVTX "Whether to build with the NVTX library" ON)
option(K2_ENABLE_BENCHMARK "Whether to enable benchmark" ON)

if(NOT K2_USE_PYTORCH)
  message(FATAL_ERROR "\
    Please set K2_USE_PYTORCH to ON.
    Support for other frameworks will be added later")
endif()

# To consist with TorchConfig.cmake from pip
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")

if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY) # `poetry build` use built-in CMAKE_LIBRARY_OUTPUT_DIRECTORY
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
endif()

if(WIN32 AND BUILD_SHARED_LIBS)
  message(STATUS "Set BUILD_SHARED_LIBS to OFF for Windows")
  set(BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
endif()

execute_process(COMMAND
    lsb_release -sd
    OUTPUT_VARIABLE K2_OS
    OUTPUT_STRIP_TRAILING_WHITESPACE
    )
string(REGEX REPLACE "^\"+|\"+$" "" K2_OS "${K2_OS}")
message(STATUS "K2_OS: ${K2_OS}")

find_package(Git REQUIRED)
execute_process(COMMAND
    "${GIT_EXECUTABLE}" describe --always --abbrev=40
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
    OUTPUT_VARIABLE K2_GIT_SHA1
    ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE
    )
execute_process(COMMAND
    "${GIT_EXECUTABLE}" log -1 --format=%ad --date=local
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
    OUTPUT_VARIABLE K2_GIT_DATE
    ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE
    )

include(CheckIncludeFileCXX)
check_include_file_cxx(cxxabi.h K2_HAVE_CXXABI_H)
check_include_file_cxx(execinfo.h K2_HAVE_EXECINFO_H)

include(CheckCXXCompilerFlag)
check_cxx_compiler_flag("-std=c++14" K2_COMPILER_SUPPORTS_CXX14)
if(NOT K2_COMPILER_SUPPORTS_CXX14)
  message(FATAL_ERROR "
    k2 requires a compiler supporting at least C++14.
    If you are using GCC, please upgrade it to at least version 5.0.
    If you are using Clang, please upgrade it to at least version 3.4.")
endif()

set(CMAKE_CXX_STANDARD 14 CACHE STRING "The C++ version to be used.")
set(CUDAToolkit_LIBRARY_DIR "" CACHE STRING "The cudatoolkit library directory.")
set(CUDAToolkit_INCLUDE_DIRS "" CACHE STRING "The cudatoolkit include directory.")

find_package(CUDAToolkit REQUIRED)
message(STATUS "found CUDAToolkit " ${CUDAToolkit_LIBRARY_DIR})
message(STATUS "CUDAToolkit_INCLUDE_DIRS " ${CUDAToolkit_INCLUDE_DIRS})
message(STATUS "CUDAToolkit_LIBRARY_DIR " ${CUDAToolkit_LIBRARY_DIR})

# With many architectures set here, the nvcc build time would be much longer.
# Thus, "61" is put here for speed and compatibility.
# @ToDo Need to cover more architectures, before release these code.
set(CMAKE_CUDA_ARCHITECTURES 61)
set(CMAKE_CUDA_FLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_CUDA_FLAGS} --expt-extended-lambda ")
set(CMAKE_CUDA_STANDARD ${CMAKE_CXX_STANDARD})

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Modules)
find_package(Valgrind)
if(Valgrind_FOUND)
  find_program(MEMORYCHECK_COMMAND NAMES ${Valgrind_EXECUTABLE})
  set(MEMORYCHECK_COMMAND_OPTIONS "--suppressions=${CMAKE_SOURCE_DIR}/scripts/valgrind.supp --leak-check=full")
  include(Dart)
  message(STATUS "To check memory, run `ctest -R <NAME> -D ExperimentalMemCheck`")
endif()
enable_testing()

# Find python
set(Python3_FIND_VIRTUALENV FIRST)
find_package(Python3 COMPONENTS Interpreter Development REQUIRED)
set(PYTHON_EXECUTABLE ${Python3_EXECUTABLE})
message(STATUS "Set PYTHON_EXECUTABLE = ${Python3_EXECUTABLE}")

# Find c++ deps
if(K2_USE_PYTORCH)
  add_definitions(-DK2_USE_PYTORCH)
  add_definitions(-DTORCH_API_INCLUDE_EXTENSION_H)
  set(Torch_DIR "${Python3_SITELIB}/torch/share/cmake/Torch")
  find_package(Torch REQUIRED)
endif()
find_package(pybind11 REQUIRED)
if(CUDA_VERSION VERSION_LESS 11.0)
  # CUB is included in CUDA toolkit 11.0 and above
find_package(cub REQUIRED)
endif()

find_package(moderngpu REQUIRED)
find_package(GTest REQUIRED)

set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --compiler-options -Wall --compiler-options -Wno-unknown-pragmas")
add_subdirectory(k2)
