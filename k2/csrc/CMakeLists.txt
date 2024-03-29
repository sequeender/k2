# Make project root dir as the INCLUDE_DIRECTORIES of all tagets in csrc/.
# The header should be included as `#include "k2/csrc/.."`, to avoid conflicts.
include_directories(${CMAKE_SOURCE_DIR})

add_subdirectory(host)

#---------------------------- Build K2 CUDA sources ----------------------------

configure_file(version.h.in ${CMAKE_CURRENT_BINARY_DIR}/version.h @ONLY)
message(STATUS "Generated ${CMAKE_CURRENT_BINARY_DIR}/version.h")

add_library(k2_log log.cu)
if(K2_HAVE_EXECINFO_H)
  target_compile_definitions(k2_log PRIVATE K2_HAVE_EXECINFO_H=1)
endif()

if(K2_HAVE_CXXABI_H)
  target_compile_definitions(k2_log PRIVATE K2_HAVE_CXXABI_H=1)
endif()

add_library(k2_nvtx INTERFACE)
target_include_directories(k2_nvtx INTERFACE ${CMAKE_SOURCE_DIR})
if(K2_ENABLE_NVTX)
  target_compile_definitions(k2_nvtx INTERFACE K2_ENABLE_NVTX=1)
endif()

# please keep it sorted
set(context_srcs
  algorithms.cu
  array_ops.cu
  context.cu
  cudpp/cudpp.cu
  dtype.cu
  fsa.cu
  fsa_algo.cu
  fsa_utils.cu
  hash.cu
  host_shim.cu
  intersect.cu
  intersect_dense.cu
  intersect_dense_pruned.cu
  math.cu
  moderngpu_allocator.cu
  pinned_context.cu
  ragged.cu
  ragged_ops.cu
  ragged_utils.cu
  rand.cu
  rm_epsilon.cu
  tensor.cu
  tensor_ops.cu
  thread_pool.cu
  timer.cu
  top_sort.cu
  utils.cu
)

if(K2_USE_PYTORCH)
  list(APPEND context_srcs pytorch_context.cu)
else()
  list(APPEND context_srcs default_context.cu)
endif()

# the target
add_library(k2_context SHARED ${context_srcs})
target_compile_definitions(k2_context PUBLIC K2_TORCH_VERSION_MAJOR=${K2_TORCH_VERSION_MAJOR})
target_compile_definitions(k2_context PUBLIC K2_TORCH_VERSION_MINOR=${K2_TORCH_VERSION_MINOR})

set_target_properties(k2_context PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

# lib deps
if(CUDA_VERSION VERSION_LESS 11.0)
  target_link_libraries(k2_context PUBLIC cub::cub)
endif()
target_link_libraries(k2_context PUBLIC fsa k2_log k2_nvtx moderngpu::moderngpu)
if(K2_USE_PYTORCH)
  target_link_libraries(k2_context PUBLIC ${TORCH_LIBRARIES})
  target_link_libraries(k2_context PUBLIC ${Python3_SITELIB}/torch/lib/libtorch_python.so)
endif()
target_include_directories(context PUBLIC ${PYTHON_INCLUDE_DIRS})
add_definitions(-DTORCH_API_INCLUDE_EXTENSION_H)

#---------------------------- Test K2 CUDA sources ----------------------------

add_library(test_utils SHARED test_utils.cu)
target_link_libraries(test_utils PUBLIC
    k2_context
    GTest::gtest)

# please sort the source files alphabetically
set(cuda_tests
  algorithms_test
  array_ops_test
  array_test
  fsa_algo_test
  fsa_test
  fsa_utils_test
  hash_test
  host_shim_test
  intersect_test
  log_test
  macros_test
  math_test
  nvtx_test
  pinned_context_test
  ragged_shape_test
  ragged_test
  ragged_utils_test
  rand_test
  rm_epsilon_test
  tensor_ops_test
  tensor_test
  thread_pool_test
  top_sort_test
  utils_test
)

# utility function to add gtest
function(k2_add_cuda_test name)
  # TODO(haowen): add prefix `cu` for now to avoid name conflicts
  # with files in k2/csrc/, will remove this finally.
  set(target_name "cu_${name}")
  add_executable(${target_name} "${name}.cu")
  set_target_properties(${target_name} PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
  target_link_libraries(${target_name}
    PRIVATE
    k2_context
    fsa  # for code in k2/csrc/host
    GTest::gtest
    GTest::gtest_main
    test_utils
  )
  add_test(NAME "Test.Cuda.${target_name}"
    COMMAND
    $<TARGET_FILE:${target_name}>
  )
endfunction()

foreach(name IN LISTS cuda_tests)
  k2_add_cuda_test(${name})
endforeach()

if(K2_ENABLE_BENCHMARK)
  add_subdirectory(benchmark)
endif()
