cmake_minimum_required(VERSION 3.3) # support visibility settings

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/Modules")

project(thread-pool-cpp VERSION 1.1.0)

option(THREAD_POOL_CPP_BUILD_TESTS "Build tests." OFF)
option(THREAD_POOL_CPP_BUILD_BENCHMARKS "Build benchmarks." OFF)
option(THREAD_POOL_CPP_INSTALL "Install library." ON)

if(THREAD_POOL_CPP_BUILD_TESTS)
  include(CTest)
endif()

# If true C++11 thread_local support exists, we will use it:
include(thread_pool_has_thread_local_storage)
thread_pool_has_thread_local_storage(THREAD_POOL_HAS_THREAD_LOCAL_STORAGE)
message("THREAD_POOL_HAS_THREAD_LOCAL_STORAGE: ${THREAD_POOL_HAS_THREAD_LOCAL_STORAGE}")

if(NOT THREAD_POOL_HAS_THREAD_LOCAL_STORAGE)
  # Else, we will check for backups
  include(thread_pool_has_thread_storage)
  thread_pool_has_thread_storage(THREAD_POOL_HAS_THREAD_STORAGE)
  message("THREAD_POOL_HAS_THREAD_STORAGE: ${THREAD_POOL_HAS_THREAD_STORAGE}")
  if(NOT THREAD_POOL_HAS_THREAD_STORAGE)
    message(FATAL_ERROR "Compiler does not support: thread_local, __thread, or declspec(thread)")
  endif()
endif()

# http://stackoverflow.com/a/11583676
add_library(thread-pool-cpp INTERFACE)

# Variable used in: benchmark, tests
set(THREAD_POOL_CPP_INC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_include_directories(thread-pool-cpp
  INTERFACE
  "$<BUILD_INTERFACE:${THREAD_POOL_CPP_INC_DIR}>"
  "$<INSTALL_INTERFACE:include>"
  )

option(THREAD_POOL_CPP_HAS_CPP14 "Has true C++14 support." ON)

if(THREAD_POOL_HAS_CPP14)
  set(CMAKE_CXX_STANDARD 14)
else()
  set(CMAKE_CXX_STANDARD 11)
endif()

target_compile_features(
  thread-pool-cpp
  INTERFACE
  cxx_auto_type
  cxx_nullptr
  cxx_lambdas
  cxx_lambda_init_captures
  )

message("CXX_STANDARD is .......... ${CXX_STANDARD}")

if(THREAD_POOL_CPP_BUILD_TESTS)
  add_subdirectory(tests)
endif()

if(THREAD_POOL_CPP_BUILD_BENCHMARKS)
  add_subdirectory(benchmark)
endif()

if(NOT THREAD_POOL_CPP_INSTALL)
  return()
endif()

#### install

set(TARGET_NAME thread-pool-cpp)

set(config_install_dir "lib/cmake/${TARGET_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Configuration
set(version_config "${generated_dir}/${TARGET_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${TARGET_NAME}Config.cmake")
set(targets_export_name "${TARGET_NAME}Targets")
set(namespace "${TARGET_NAME}::")

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
  "${version_config}" COMPATIBILITY SameMajorVersion
  )

configure_package_config_file(
  "cmake/Config.cmake.in"
  "${project_config}"
  INSTALL_DESTINATION "${config_install_dir}"
  )

install(
  TARGETS thread-pool-cpp
  EXPORT "${targets_export_name}"
  LIBRARY DESTINATION "lib"
  ARCHIVE DESTINATION "lib"
  RUNTIME DESTINATION "bin"
  INCLUDES DESTINATION "${include_install_dir}"
  )

install(
  DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/thread_pool
  DESTINATION "${include_install_dir}" 
  FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
  )

install(
  FILES "${project_config}" "${version_config}"
  DESTINATION "${config_install_dir}"
  )

install(
  EXPORT "${targets_export_name}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}"
  )
