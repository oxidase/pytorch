cmake_minimum_required(VERSION 3.5 FATAL_ERROR)
#cmake_policy(SET CMP0022 NEW)
#cmake_policy(SET CMP0023 NEW)

# ---[ Project and semantic versioning.
project(Caffe2 CXX C)

set(CAFFE2_VERSION_MAJOR 0)
set(CAFFE2_VERSION_MINOR 8)
set(CAFFE2_VERSION_PATCH 2)
set(CAFFE2_VERSION
    "${CAFFE2_VERSION_MAJOR}.${CAFFE2_VERSION_MINOR}.${CAFFE2_VERSION_PATCH}")

# One variable that determines whether the current cmake process is being run
# with the main Caffe2 library. This is useful for building modules - if
# modules are built with the main Caffe2 library then one does not need to do
# find caffe2 in the cmake script. One can usually guard it in some way like
#    if (NOT CAFFE2_CMAKE_BUILDING_WITH_MAIN_REPO)
#      find_package(Caffe2 REQUIRED)
#    endif()
set(CAFFE2_CMAKE_BUILDING_WITH_MAIN_REPO ON)

if(NOT DEFINED BLAS_SET_BY_USER)
  if(DEFINED BLAS)
    set(BLAS_SET_BY_USER TRUE)
  else()
    message(STATUS "Not forcing any particular BLAS to be found")
    set(BLAS_SET_BY_USER FALSE)
  endif()
  set(BLAS_SET_BY_USER ${BLAS_SET_BY_USER} CACHE STRING "Marks whether BLAS was manually set by user or auto-detected")
endif()

# These lines are an attempt to make find_package(cuda) pick up
# libcuda.dylib, and not cuda.framework.  It doesn't work all
# the time, but it seems to help for some users.
# TODO: replace this with a more robust fix
if(APPLE)
  set(CMAKE_FIND_FRAMEWORK LAST)
  set(CMAKE_FIND_APPBUNDLE LAST)
endif()

# Get clang version on macOS
if(APPLE)
  EXECUTE_PROCESS( COMMAND ${CMAKE_CXX_COMPILER} --version OUTPUT_VARIABLE clang_full_version_string )
  string(REGEX REPLACE "Apple LLVM version ([0-9]+\\.[0-9]+).*" "\\1" CLANG_VERSION_STRING ${clang_full_version_string})
  MESSAGE( STATUS "CLANG_VERSION_STRING:         " ${CLANG_VERSION_STRING} )
endif()

# ---[ Options.
# Note to developers: if you add an option below, make sure you also add it to
# cmake/Summary.cmake so that the summary prints out the option values.
include(CMakeDependentOption)
option(BUILD_CAFFE2 "Build Caffe2" ON)
option(BUILD_ATEN "Build ATen" OFF)
option(BUILD_BINARY "Build C++ binaries" ON)
option(BUILD_DOCS "Build Caffe2 documentation" OFF)
option(BUILD_CUSTOM_PROTOBUF "Build and use Caffe2's own protobuf under third_party" ON)
option(BUILD_PYTHON "Build Python binaries" ON)
option(BUILD_SHARED_LIBS "Build libcaffe2.so" ON)
cmake_dependent_option(
    CAFFE2_LINK_LOCAL_PROTOBUF "If set, build protobuf inside libcaffe2.so." ON
    "BUILD_SHARED_LIBS AND BUILD_CUSTOM_PROTOBUF" OFF)
cmake_dependent_option(
    CAFFE2_USE_MSVC_STATIC_RUNTIME "Using MSVC static runtime libraries" ON
    "NOT BUILD_SHARED_LIBS" OFF)
cmake_dependent_option(
    BUILD_TEST "Build Caffe2 C++ test binaries (need gtest and gbenchmark)" OFF
    "BUILD_CAFFE2" OFF)
cmake_dependent_option(
    INSTALL_TEST "Install test binaries if BUILD_TEST is on" OFF
    "BUILD_TEST" OFF)
option(USE_ACL "Use ARM Compute Library" OFF)
option(USE_ASAN "Use Address Sanitizer" OFF)
option(USE_ATEN "Use ATen" OFF)
option(USE_CUDA "Use CUDA" ON)
option(USE_ROCM "Use ROCm" OFF)
option(CAFFE2_STATIC_LINK_CUDA "Statically link CUDA libraries" OFF)
cmake_dependent_option(
    USE_CUDNN "Use cuDNN" ON
    "USE_CUDA" OFF)
option(USE_FFMPEG "Use ffmpeg" OFF)
cmake_dependent_option(
    USE_GFLAGS "Use GFLAGS" ON
    "BUILD_CAFFE2" OFF)
cmake_dependent_option(
    USE_GLOG "Use GLOG" ON
    "BUILD_CAFFE2" OFF)
cmake_dependent_option(
    USE_GLOO "Use Gloo" ON
    "BUILD_CAFFE2" OFF)
option(USE_GLOO_IBVERBS "Use Gloo IB verbs for distributed support" OFF)
cmake_dependent_option(
    USE_LEVELDB "Use LEVELDB" ON
    "BUILD_CAFFE2" OFF)
option(USE_LITE_PROTO "Use lite protobuf instead of full." OFF)
cmake_dependent_option(
    USE_LMDB "Use LMDB" ON
    "BUILD_CAFFE2" OFF)
cmake_dependent_option(
    USE_METAL "Use Metal for iOS build" ON
    "BUILD_CAFFE2" OFF)
cmake_dependent_option(
    USE_MOBILE_OPENGL "Use OpenGL for mobile code" ON
    "BUILD_CAFFE2" OFF)
cmake_dependent_option(
    USE_MPI "Use MPI" ON
    "BUILD_CAFFE2" OFF)
option(USE_NATIVE_ARCH "Use -march=native" OFF)
option(USE_NCCL "Use NCCL" ON)
option(USE_SYSTEM_NCCL "Use system-wide NCCL" OFF)
option(USE_NERVANA_GPU "Use Nervana GPU backend" OFF)
option(USE_NNAPI "Use NNAPI" OFF)
option(USE_NNPACK "Use NNPACK" ON)
option(USE_NUMA "Use NUMA (only available on Linux)" ON)
cmake_dependent_option(
    USE_NVRTC "Use NVRTC. Only available if USE_CUDA is on." OFF
    "USE_CUDA" OFF)
option(USE_OBSERVERS "Use observers module." OFF)
option(USE_OPENCL "Use OpenCL" OFF)
cmake_dependent_option(
    USE_OPENCV "Use OpenCV" ON
    "BUILD_CAFFE2" OFF)
option(USE_OPENMP "Use OpenMP for parallel code" OFF)
option(USE_PROF "Use profiling" OFF)
option(USE_REDIS "Use Redis" OFF)
option(USE_ROCKSDB "Use RocksDB" OFF)
option(USE_SNPE "Use Qualcomm's SNPE library" OFF)
option(USE_TENSORRT "Using Nvidia TensorRT library" OFF)
option(USE_ZMQ "Use ZMQ" OFF)
option(USE_ZSTD "Use ZSTD" OFF)
option(USE_MKLDNN "Use MKLDNN" OFF)
cmake_dependent_option(
  USE_IDEEP "Use IDEEP interface in MKL BLAS" ON
    "BUILD_CAFFE2" OFF)
cmake_dependent_option(
  USE_MKLML "Use MKLML interface in MKL BLAS" ON
    "BUILD_CAFFE2" OFF)
option(USE_DISTRIBUTED "Use THD (distributed)" OFF)
option(USE_DISTRIBUTED_MW "Use THD (distributed) master worker" OFF)

if (USE_ATEN)
    set(BUILD_ATEN ${USE_ATEN})
endif()

# ---[ CMake scripts + modules
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)

if (MSVC AND ${BUILD_SHARED_LIBS})
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

# ---[ CMake build directories
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

enable_testing()

# ---[ Build variables set within the cmake tree
include(cmake/BuildVariables.cmake)
set(CAFFE2_WHITELIST "" CACHE STRING "A whitelist file of files that one should build.")

# Set default build type
if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "Build type not set - defaulting to Release")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build from: Debug Release RelWithDebInfo MinSizeRel Coverage." FORCE)
endif()

# ---[ Misc checks to cope with various compiler modes
include(cmake/MiscCheck.cmake)

# External projects
include(ExternalProject)

# ---[ Utils
# TODO: merge the following 3 files into cmake/public/utils.cmake.
include(cmake/Utils.cmake)
include(cmake/public/utils.cmake)

# ---[ Dependencies
include(cmake/Dependencies.cmake)

# ---[ Whitelist file if whitelist is specified
include(cmake/Whitelist.cmake)

# ---[ Set link flag, handle additional deps for gcc 4.8 and above
if(CMAKE_COMPILER_IS_GNUCXX AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.8.0 AND NOT ANDROID)
  message(STATUS "GCC ${CMAKE_CXX_COMPILER_VERSION}: Adding gcc and gcc_s libs to link line")
  list(APPEND Caffe2_DEPENDENCY_LIBS gcc_s gcc)
endif()

# ---[ Build flags
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 11)
if(NOT MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -fPIC")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-narrowing")
  # Eigen fails to build with some versions, so convert this to a warning
  # Details at http://eigen.tuxfamily.org/bz/show_bug.cgi?id=1459
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-invalid-partial-specialization")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wextra")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-missing-field-initializers")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-type-limits")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-typedef-redefinition")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unknown-warning-option")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unknown-pragmas")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-sign-compare")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-parameter")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-variable")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-function")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-private-field")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-result")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-inconsistent-missing-override")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-aligned-allocation-unavailable")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error=deprecated-declarations")
  if ((APPLE AND (NOT ("${CLANG_VERSION_STRING}" VERSION_LESS "9.0")))
    OR (CMAKE_COMPILER_IS_GNUCXX AND (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 7.0)))
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -faligned-new")
  endif()
  if ($ENV{WERROR})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
  endif($ENV{WERROR})
else()
  foreach(flag_var
      CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
      CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
    if (${CAFFE2_USE_MSVC_STATIC_RUNTIME})
      if(${flag_var} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
      endif(${flag_var} MATCHES "/MD")
    else()
      if(${flag_var} MATCHES "/MT")
        string(REGEX REPLACE "/MT" "/MD" ${flag_var} "${${flag_var}}")
      endif()
    endif()
    # /bigobj increases number of sections in .obj file, which is needed to link
    # against libaries in Python 2.7 under Windows
    set(${flag_var} "${${flag_var}} /MP /bigobj")
  endforeach(flag_var)
endif()

set (CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fno-omit-frame-pointer -O0")
set (CMAKE_LINKER_FLAGS_DEBUG "${CMAKE_STATIC_LINKER_FLAGS_DEBUG} -fno-omit-frame-pointer -O0")
if (USE_ASAN)
    set (CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=address")
    set (CMAKE_LINKER_FLAGS_DEBUG "${CMAKE_STATIC_LINKER_FLAGS_DEBUG} -fsanitize=address")
endif()

if(ANDROID)
  if(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s")
  else()
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -s")
  endif()
endif()

if(NOT APPLE AND UNIX)
  list(APPEND Caffe2_DEPENDENCY_LIBS dl)
endif()

# Prefix path to Caffe2 headers.
# If a directory containing installed Caffe2 headers was inadvertently
# added to the list of include directories, prefixing
# PROJECT_SOURCE_DIR means this source tree always takes precedence.
include_directories(BEFORE ${PROJECT_SOURCE_DIR})

# Prefix path to generated Caffe2 headers.
# These need to take precedence over their empty counterparts located
# in PROJECT_SOURCE_DIR.
include_directories(BEFORE ${PROJECT_BINARY_DIR})

# ---[ Old caffe protobuf
if(BUILD_CAFFE2)
  add_subdirectory(caffe/proto)
endif()

# ---[ Shared build
add_subdirectory(c10)

# ---[ Main build
add_subdirectory(caffe2)

# --[ Documentation
if(BUILD_DOCS)
  # check if Doxygen is installed
  find_package(Doxygen)
  if (DOXYGEN_FOUND)
    message("Generating documentation")

    set(DOXYGEN_C_IN ${CMAKE_CURRENT_SOURCE_DIR}/docs/caffe2/.Doxyfile-c)
    set(DOXYGEN_C_OUT ${CMAKE_CURRENT_SOURCE_DIR}/docs/caffe2/Doxyfile-c)
    set(DOXYGEN_P_IN ${CMAKE_CURRENT_SOURCE_DIR}/docs/caffe2/.Doxyfile-python)
    set(DOXYGEN_P_OUT ${CMAKE_CURRENT_SOURCE_DIR}/docs/caffe2/Doxyfile-python)

    if(EXISTS ${CMAKE_CURRENT_BINARY_DIR}/docs)
      file(REMOVE_RECURSE ${CMAKE_CURRENT_BINARY_DIR}/docs)
    endif (EXISTS ${CMAKE_CURRENT_BINARY_DIR}/docs)

    file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs)
    configure_file(${DOXYGEN_C_IN} ${DOXYGEN_C_OUT} @ONLY)
    configure_file(${DOXYGEN_P_IN} ${DOXYGEN_P_OUT} @ONLY)

    add_custom_target(doc_doxygen_c ALL
        COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_C_OUT}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Generating C++ API documentation with Doxygen"
        VERBATIM)

    add_custom_target(doc_doxygen_python ALL
        COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_P_OUT}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Generating Python API documentation with Doxygen"
        VERBATIM)
  else (DOXYGEN_FOUND)
    message(FATAL_ERROR "Doxygen needs to be installed to generate the documentation")
  endif (DOXYGEN_FOUND)
endif (BUILD_DOCS)

# ---[ CMake related files
# Uninistall option.
if(NOT TARGET caffe2_uninstall)
  configure_file(
      ${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in
      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
      IMMEDIATE @ONLY)

  add_custom_target(caffe2_uninstall
      COMMAND ${CMAKE_COMMAND} -P
      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif()

# ---[ Make configuration files for cmake to allow dependent libraries
# easier access to Caffe2.

if ((NOT USE_GLOG) OR (NOT USE_GFLAGS) OR BUILD_CUSTOM_PROTOBUF)
  message(WARNING
      "Generated cmake files are only fully tested if one builds "
      "with system glog, gflags, and protobuf. Other settings may "
      "generate files that are not well tested.")
endif()

if (USE_CUDA OR USE_ROCM)
  # TODO: check if we should include other cuda dependency libraries
  # to the interface as well.

endif()

# Note(jiayq): when building static libraries, all PRIVATE dependencies
# will also become interface libraries, and as a result if there are any
# dependency libraries that are not exported, the following install export
# script will fail. As a result, we will only provide the targets cmake
# files for shared lib installation. For more info, read:
# https://cmake.org/pipermail/cmake/2016-May/063400.html
if (BUILD_SHARED_LIBS)
  configure_file(
      ${PROJECT_SOURCE_DIR}/cmake/Caffe2ConfigVersion.cmake.in
      ${PROJECT_BINARY_DIR}/Caffe2ConfigVersion.cmake
      @ONLY)
  configure_file(
      ${PROJECT_SOURCE_DIR}/cmake/Caffe2Config.cmake.in
      ${PROJECT_BINARY_DIR}/Caffe2Config.cmake
      @ONLY)
  install(FILES
      ${PROJECT_BINARY_DIR}/Caffe2ConfigVersion.cmake
      ${PROJECT_BINARY_DIR}/Caffe2Config.cmake
      DESTINATION share/cmake/Caffe2
      COMPONENT dev)
  install(FILES
      ${PROJECT_SOURCE_DIR}/cmake/public/cuda.cmake
      ${PROJECT_SOURCE_DIR}/cmake/public/glog.cmake
      ${PROJECT_SOURCE_DIR}/cmake/public/gflags.cmake
      ${PROJECT_SOURCE_DIR}/cmake/public/protobuf.cmake
      ${PROJECT_SOURCE_DIR}/cmake/public/threads.cmake
      ${PROJECT_SOURCE_DIR}/cmake/public/utils.cmake
      DESTINATION share/cmake/Caffe2/public
      COMPONENT dev)
  install(DIRECTORY
      ${PROJECT_SOURCE_DIR}/cmake/Modules_CUDA_fix
      DESTINATION share/cmake/Caffe2/
      COMPONENT dev)
  install(EXPORT Caffe2Targets DESTINATION share/cmake/Caffe2
      FILE Caffe2Targets.cmake
      COMPONENT dev)
else()
  message(WARNING
      "Generated cmake files are only available when building "
      "shared libs.")
endif()

# ---[ Modules
if (BUILD_CAFFE2)
  add_subdirectory(modules)
endif()

# ---[ Binaries
# Binaries will be built after the Caffe2 main libraries and the modules
# are built. For the binaries, they will be linked to the Caffe2 main
# libraries, as well as all the modules that are built with Caffe2 (the ones
# built in the previous Modules section above).
if (BUILD_CAFFE2)
  if (BUILD_BINARY)
    add_subdirectory(binaries)
  endif()
endif()

include(cmake/Summary.cmake)
caffe2_print_configuration_summary()
