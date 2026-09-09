# ======================================================================== #
# Copyright 2024-2025 Ingo Wald                                            #
#                                                                          #
# Licensed under the Apache License, Version 2.0 (the "License");          #
# you may not use this file except in compliance with the License.         #
# You may obtain a copy of the License at                                  #
#                                                                          #
#     http://www.apache.org/licenses/LICENSE-2.0                           #
#                                                                          #
# Unless required by applicable law or agreed to in writing, software      #
# distributed under the License is distributed on an "AS IS" BASIS,        #
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. #
# See the License for the specific language governing permissions and      #
# limitations under the License.                                           #
# ======================================================================== #

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION pybind11::thin_lto)

#find_package(anari)

set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQURIED ON)

pybind11_add_module(pynari
  # 'thin' objects that only handle the reference without any other
  # operations on it:
  Material.h
  Material.cpp
  Light.h
  Light.cpp
  World.h
  World.cpp
  Volume.h
  Group.h
  Geometry.h
  Geometry.cpp
  Camera.h
  Renderer.h
  Surface.h
  Surface.cpp
  SpatialField.h
  Instance.h
  Instance.cpp

  # more complicted objects that require additional functions
  Group.h
  Group.cpp
  Context.h
  Context.cpp
  Object.h
  Object.cpp
  Device.h
  Device.cpp
  Array.h
  Array.cpp
  Frame.h
  Frame.cpp
  Sampler.h
  Sampler.cpp
  
  # the actual pybind11 bindings file
  bindings.cpp
#  dummy.cu
)
if (PYNARI_HAVE_CUDA)
  message("setting language to CUDA")
  set_source_files_properties(Context.cpp
    PROPERTIES 
    LANGUAGE CUDA
    )
endif()

set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)
target_include_directories(pynari PUBLIC ${PROJECT_SOURCE_DIR})
set_target_properties(pynari PROPERTIES
  PYTHON_MODULE_NAME "pynari"
  )
target_link_libraries(pynari PUBLIC
  anari::anari
  )
if (CMAKE_CUDA_ARCHITECTURES)
  set_target_properties(pynari PROPERTIES
    CUDA_ARCHITECTURES ${CMAKE_CUDA_ARCHITECTURES}
    )
else()
  set_target_properties(pynari PROPERTIES
    CUDA_ARCHITECTURES all-major
    )
endif()

if (PYNARI_HAVE_CUDA)
  set_source_files_properties(Context.cpp
    PROPERTIES
    LANGUAGE CUDA
    )
  set_source_files_properties(Frame.cpp
    PROPERTIES
    LANGUAGE CUDA
    )
  target_compile_definitions(pynari PRIVATE -DPYNARI_HAVE_CUDA=1)
  set_target_properties(pynari PROPERTIES
    CUDA_SEPARABLE_COMPILATION ON
    POSITION_INDEPENDENT_CODE ON
    CUDA_RESOLVE_DEVICE_SYMBOLS ON
    CUDA_USE_STATIC_CUDA_RUNTIME ON
    BUILD_WITH_INSTALL_RPATH ON
    )
endif()

target_link_libraries(pynari
  PUBLIC
#  barney::anari_library_barney
  anari::helium
  anari::anari
  )

# When built through pip (scikit-build-core), CMake is configured with
# SKBUILD defined and the install prefix pointing at the wheel's platlib
# directory. In that case install pynari as a proper python package:
#
#   <platlib>/pynari/__init__.py
#   <platlib>/pynari/pynari.<abi>.so
#
# The same layout can also be requested for a plain cmake build with
# -DPYNARI_BUILD_FOR_PIP=ON.
if (DEFINED SKBUILD)
  set(PYNARI_BUILD_FOR_PIP_DEFAULT ON)
else()
  set(PYNARI_BUILD_FOR_PIP_DEFAULT OFF)
endif()
option(PYNARI_BUILD_FOR_PIP
  "Install pynari as a python package directory (pynari/__init__.py + module)"
  ${PYNARI_BUILD_FOR_PIP_DEFAULT})
option(PYNARI_BUNDLE_ANARI
  "When building for pip, copy the ANARI runtime library into the package"
  OFF)
option(PYNARI_BUNDLE_BARNEY
  "When building for pip, copy the barney runtime libraries into the package"
  OFF)
message(STATUS "PYNARI_BUILD_FOR_PIP ${PYNARI_BUILD_FOR_PIP}")

if (PYNARI_BUILD_FOR_PIP)
  message(STATUS "building pynari as a python package...")
  # Look for shared libraries next to the module first (bundled libs),
  # then in the directories they were linked from (non-bundled libs,
  # e.g. an ANARI SDK installed in a non-system prefix).
  if (APPLE)
    set(PYNARI_RPATH_ORIGIN "@loader_path")
  else()
    set(PYNARI_RPATH_ORIGIN "$ORIGIN")
  endif()
  set_target_properties(pynari PROPERTIES
    INSTALL_RPATH "${PYNARI_RPATH_ORIGIN}"
    INSTALL_RPATH_USE_LINK_PATH ON
    )
  # destinations are relative to CMAKE_INSTALL_PREFIX, which
  # scikit-build-core sets to the wheel's platlib directory
  install(
    TARGETS pynari
    LIBRARY DESTINATION pynari
    RUNTIME DESTINATION pynari
    )
  install(
  # note(iw): the reason we store this as 'becomes.__init__.py'
  # in the source tree is because we want this file to be
  # 'recognized' by python _only_ when installed through a pip
  # package. If it is called __init__.py in the source
  # tree, python will try to call it even if we run on a
  # local build that's not a pip package, which will
  # fail.
    FILES becomes.__init__.py
    RENAME __init__.py
    # where to put it - we want this in <install>/pynari
    # $(install_dir)/<packagename(=pynari)>/__init__.py
    DESTINATION pynari
    )
  if (PYNARI_BUNDLE_ANARI)
    install(
      IMPORTED_RUNTIME_ARTIFACTS anari::anari
      LIBRARY DESTINATION pynari
      RUNTIME DESTINATION pynari
      )
  endif()
  if (PYNARI_BUNDLE_BARNEY)
    find_package(barney REQUIRED)
    install(
      IMPORTED_RUNTIME_ARTIFACTS
      barney::barney
      barney::anari_library_barney
      LIBRARY DESTINATION pynari
      RUNTIME DESTINATION pynari
      )
  endif()
else()
  include(GNUInstallDirs)
  install(
    IMPORTED_RUNTIME_ARTIFACTS pynari
    
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
 #   NAMELINK_SKIP
    # on Windows put the dlls into bin
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    # ... and the import lib into the devel package
#    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )
  install(
    TARGETS pynari
#    NAMELINK_SKIP
    # on Windows put the dlls into bin
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    # ... and the import lib into the devel package
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()
