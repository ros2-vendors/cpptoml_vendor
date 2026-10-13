cmake_minimum_required(VERSION 3.7)

project(cpptoml_vendor)

find_package(ament_cmake REQUIRED)

# cpptoml is a header-only C++ TOML parser (skystrife/cpptoml). iceoryx_posh
# normally downloads + builds it inline via an ExternalProject (DOWNLOAD_TOML_LIB=ON),
# but on the VxWorks SDK that bundled path runs host tools (git, doxygen) that fail
# when the ROS 2 install lib dir is on LD_LIBRARY_PATH. Instead we build cpptoml
# once as a vendor and install its cmake config, so iceoryx can consume it via
# find_package(cpptoml) with DOWNLOAD_TOML_LIB=OFF (no inline download/build).
#
# Version 0.1.1 matches the tag iceoryx pins. Header-only: cross-compiling is a
# no-op; we build it (to get its installed cmake config exporting the `cpptoml`
# INTERFACE target) and stage the install tree.

option(FORCE_BUILD_VENDOR_PKG "Build cpptoml from source" OFF)

if(NOT FORCE_BUILD_VENDOR_PKG)
  find_package(cpptoml QUIET CONFIG)
endif()

macro(build_cpptoml)
  set(extra_cmake_args)
  if(DEFINED CMAKE_BUILD_TYPE)
    list(APPEND extra_cmake_args -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})
  endif()
  if(DEFINED CMAKE_TOOLCHAIN_FILE)
    list(APPEND extra_cmake_args "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}")
  endif()
  # cpptoml's own build options: no examples, and do not look for the host
  # Doxygen (its docs target is irrelevant and the host doxygen is unusable here).
  list(APPEND extra_cmake_args
    -DCPPTOML_BUILD_EXAMPLES=OFF
    -DENABLE_LIBCXX=OFF
    -DCMAKE_DISABLE_FIND_PACKAGE_Doxygen=TRUE)

  include(ExternalProject)

  externalproject_add(cpptoml_external
    GIT_REPOSITORY https://github.com/skystrife/cpptoml.git
    GIT_TAG v0.1.1
    TIMEOUT 600
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_install
      ${extra_cmake_args}
  )

  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_install/
    DESTINATION
      ${CMAKE_INSTALL_PREFIX}
    USE_SOURCE_PERMISSIONS
  )
endmacro()

if(NOT cpptoml_FOUND)
  message(STATUS "Did not find cpptoml, building from source (v0.1.1)")
  build_cpptoml()
endif()

ament_package()
