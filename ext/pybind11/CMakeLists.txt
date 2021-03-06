# CMakeLists.txt -- Build system for the pybind11 modules
#
# Copyright (c) 2015 Wenzel Jakob <wenzel@inf.ethz.ch>
#
# All rights reserved. Use of this source code is governed by a
# BSD-style license that can be found in the LICENSE file.

cmake_minimum_required(VERSION 2.8.12)

project(pybind11)

# Check if pybind11 is being used directly or via add_subdirectory
set(PYBIND11_MASTER_PROJECT OFF)
if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  set(PYBIND11_MASTER_PROJECT ON)
endif()

option(PYBIND11_INSTALL "Install pybind11 header files?" ${PYBIND11_MASTER_PROJECT})
option(PYBIND11_TEST    "Build pybind11 test suite?"     ${PYBIND11_MASTER_PROJECT})
option(PYBIND11_WERROR  "Report all warnings as errors"  OFF)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/tools")

include(pybind11Tools)

# Cache variables so pybind11_add_module can be used in parent projects
set(PYBIND11_INCLUDE_DIR "${CMAKE_CURRENT_LIST_DIR}/include" CACHE INTERNAL "")
set(PYTHON_INCLUDE_DIRS ${PYTHON_INCLUDE_DIRS} CACHE INTERNAL "")
set(PYTHON_LIBRARIES ${PYTHON_LIBRARIES} CACHE INTERNAL "")
set(PYTHON_MODULE_PREFIX ${PYTHON_MODULE_PREFIX} CACHE INTERNAL "")
set(PYTHON_MODULE_EXTENSION ${PYTHON_MODULE_EXTENSION} CACHE INTERNAL "")

# Compile with compiler warnings turned on
function(pybind11_enable_warnings target_name)
  if(MSVC)
    target_compile_options(${target_name} PRIVATE /W4)
  else()
    target_compile_options(${target_name} PRIVATE -Wall -Wextra -Wconversion)
  endif()

  if(PYBIND11_WERROR)
    if(MSVC)
      target_compile_options(${target_name} PRIVATE /WX)
    else()
      target_compile_options(${target_name} PRIVATE -Werror)
    endif()
  endif()
endfunction()

set(PYBIND11_HEADERS
  include/pybind11/attr.h
  include/pybind11/cast.h
  include/pybind11/chrono.h
  include/pybind11/common.h
  include/pybind11/complex.h
  include/pybind11/descr.h
  include/pybind11/options.h
  include/pybind11/eigen.h
  include/pybind11/eval.h
  include/pybind11/functional.h
  include/pybind11/numpy.h
  include/pybind11/operators.h
  include/pybind11/pybind11.h
  include/pybind11/pytypes.h
  include/pybind11/stl.h
  include/pybind11/stl_bind.h
  include/pybind11/typeid.h
)
string(REPLACE "include/" "${CMAKE_CURRENT_SOURCE_DIR}/include/"
       PYBIND11_HEADERS "${PYBIND11_HEADERS}")

if (PYBIND11_TEST)
  add_subdirectory(tests)
endif()

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# extract project version from source
file(STRINGS "${PYBIND11_INCLUDE_DIR}/pybind11/common.h" pybind11_version_defines
     REGEX "#define PYBIND11_VERSION_(MAJOR|MINOR|PATCH) ")
foreach(ver ${pybind11_version_defines})
  if (ver MATCHES "#define PYBIND11_VERSION_(MAJOR|MINOR|PATCH) +([^ ]+)$")
    set(PYBIND11_VERSION_${CMAKE_MATCH_1} "${CMAKE_MATCH_2}" CACHE INTERNAL "")
  endif()
endforeach()
set(${PROJECT_NAME}_VERSION ${PYBIND11_VERSION_MAJOR}.${PYBIND11_VERSION_MINOR}.${PYBIND11_VERSION_PATCH})

if(NOT (CMAKE_VERSION VERSION_LESS 3.0))  # CMake >= 3.0
  # Build an interface library target:
  add_library(pybind11 INTERFACE)
  target_include_directories(pybind11 INTERFACE $<BUILD_INTERFACE:${PYBIND11_INCLUDE_DIR}>
                                                $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)
  if(APPLE)
      target_link_libraries(pybind11 INTERFACE "-undefined dynamic_lookup")
  endif()
endif()

if (PYBIND11_INSTALL)
  install(FILES ${PYBIND11_HEADERS}
          DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/pybind11)
  # GNUInstallDirs "DATADIR" wrong here; CMake search path wants "share".
  set(PYBIND11_CMAKECONFIG_INSTALL_DIR "share/cmake/${PROJECT_NAME}" CACHE STRING "install path for pybind11Config.cmake")

  configure_package_config_file(tools/${PROJECT_NAME}Config.cmake.in
                                "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
                                INSTALL_DESTINATION ${PYBIND11_CMAKECONFIG_INSTALL_DIR})
  write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
                                   VERSION ${${PROJECT_NAME}_VERSION}
                                   COMPATIBILITY AnyNewerVersion)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
                ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
                tools/FindPythonLibsNew.cmake
                tools/pybind11Tools.cmake
          DESTINATION ${PYBIND11_CMAKECONFIG_INSTALL_DIR})

  if(NOT (CMAKE_VERSION VERSION_LESS 3.0))
    install(TARGETS pybind11
            EXPORT "${PROJECT_NAME}Targets")
    install(EXPORT "${PROJECT_NAME}Targets"
            NAMESPACE "${PROJECT_NAME}::"
            DESTINATION ${PYBIND11_CMAKECONFIG_INSTALL_DIR})
    message(STATUS "Exporting ${PROJECT_NAME}::pybind11 interface library target version ${${PROJECT_NAME}_VERSION}")
  endif()
endif()
