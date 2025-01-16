# Copyright (c) 2020-2024, Arm Limited and Contributors
# Copyright (c) 2024, Mobica Limited
#
# SPDX-License-Identifier: Apache-2.0
#
# Licensed under the Apache License, Version 2.0 the "License";
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

if(WIN32)
    set(CMAKE_TOOLCHAIN_FILE "C:/Program Files/Microsoft Visual Studio/2022/Community/VC/vcpkg/scripts/buildsystems/vcpkg.cmake")
    set(VCPKG_INSTALLED_ARCH_OS "${CMAKE_BINARY_DIR}/vcpkg_installed/x64-windows")
    set(VK_ADD_LAYER_PATH "${VCPKG_INSTALLED_ARCH_OS}/bin")
    message("VK_LAYER_ADD_PATH is ${VK_ADD_LAYER_PATH}")
    if(DEBUG)
        set(VKB_VALIDATION_LAYERS TRUE)
        set(VKB_VALIDATION_LAYERS_BEST_PRACTICES TRUE)
    endif()
  endif()
  if (LINUX)
    set(VCPKG_SHARE "./vcpkg_installed/x64-linux/share/")
    set(CMAKE_RC_ANDROID_TOOLCHAIN_MACHINE "~/Workspaces/Tools/vcpkg/scripts/buildsystems/vcpkg.cmake")
endif()

list(INSERT CMAKE_MODULE_PATH 0 "${CMAKE_SOURCE_DIR}/bldsys/cmake/module")

if(NOT DEFINED CMAKE_C_COMPILER_LAUNCHER AND NOT DEFINED CMAKE_CXX_COMPILER_LAUNCHER)
    find_program(CCACHE_FOUND ccache)
    find_program(SCCACHE_FOUND sccache)
    if (SCCACHE_FOUND)
        message("setting SCCACHE to ${SCCACHE_FOUND}")
        set(CMAKE_C_COMPILER_LAUNCHER ${SCCACHE_FOUND})
        set(CMAKE_CXX_COMPILER_LAUNCHER ${SCCACHE_FOUND})
    elseif(CCACHE_FOUND)
        message("setting CCACHE to ${CCACHE_FOUND}")
        set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE_FOUND})
        set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_FOUND})
    endif()
endif()

cmake_minimum_required(VERSION 3.16)

project(vulkan_samples)

# search for Vulkan SDK
find_package(Vulkan)
find_package(Ktx CONFIG REQUIRED)
find_package(directx-dxc CONFIG REQUIRED)
find_package(Catch2 CONFIG REQUIRED)
find_package(Stb REQUIRED)
find_package(spdlog CONFIG REQUIRED)

set(Vulkan_dxc_EXECUTABLE "${VCPKG_INSTALLED_ARCH_OS}/tools/directx-dxc/dxc.exe")
# message("DXC: ${Vulkan_dxc_EXECUTABLE}")

if(Vulkan_FOUND)
    find_program(Vulkan_dxc_EXECUTABLE
        NAMES dxc
        HINTS "${VCPKG_INSTALLED_ARCH_OS}/tools/directx-dxc"
    )
    if(Vulkan_dxc_EXECUTABLE)
        message(STATUS "Found DirectX Shader Compiler under ${Vulkan_dxc_EXECUTABLE}")
    else()
        message(STATUS "Couldn't find DirectX Shader Compiler executable, make sure it is present in Vulkan SDK or add it manually via Vulkan_dxc_EXECUTABLE cmake variable. HLSL shaders won't be compiled.")
    endif()
endif()

# globally add VKB_DEBUG for the debug build
add_compile_definitions($<$<CONFIG:DEBUG>:VKB_DEBUG>)

# globally set -fno-strict-aliasing, needed due to using reinterpret_cast
if (NOT MSVC)
  add_compile_options(-fno-strict-aliasing)
endif()

if(MSVC AND (DEFINED CMAKE_C_COMPILER_LAUNCHER))
    message(DEBUG "Setting MSVC flags to /Z7 for ccache compatibility.  Current flags: ${CMAKE_CXX_FLAGS_DEBUG}")
    string(REPLACE "/Zi" "/Z7" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
    string(REPLACE "/Zi" "/Z7" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
    string(REPLACE "/Zi" "/Z7" CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
    string(REPLACE "/Zi" "/Z7" CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
    message(DEBUG "New flags: ${CMAKE_CXX_FLAGS_DEBUG}")
endif()

# create output folder
file(MAKE_DIRECTORY output)

# Add path for local cmake scripts
list(APPEND CMAKE_MODULE_PATH
    ${CMAKE_CURRENT_SOURCE_DIR}/bldsys/cmake
    ${CMAKE_CURRENT_SOURCE_DIR}/bldsys/cmake/module)

include(utils)
include(global_options)
include(sample_helper)
include(check_atomic)
include(component_helper)

# Add third party libraries
add_subdirectory(third_party)

vkb__enable_testing()

# Framework v2.0 components directory
add_subdirectory(components)

# Add vulkan framework
add_subdirectory(framework)

if(VKB_BUILD_SAMPLES)
    # Add vulkan samples
    add_subdirectory(samples)
endif()

# Add vulkan app (runs all samples)
add_subdirectory(app)

add_subdirectory(antora)

# Add custom target for the Res folder
add_custom_target(copy_Res ALL
  COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/output
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/assets ${CMAKE_BINARY_DIR}/resources/assets
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/shaders ${CMAKE_BINARY_DIR}/resources/shaders
  DEPENDS
    ${CMAKE_SOURCE_DIR}/assets
    ${CMAKE_SOURCE_DIR}/shaders
)

# Add dependency to the main target
add_dependencies(${PROJECT_NAME} copy_Res)

set_target_properties(${PROJECT_NAME} PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
  LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
  ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
)


set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/release)

install(TARGETS ${PROJECT_NAME}
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION bin
  ARCHIVE DESTINATION lib
)

install(DIRECTORY ${CMAKE_BINARY_DIR}/lib DESTINATION .)
install(DIRECTORY ${CMAKE_BINARY_DIR}/bin DESTINATION .)
install(DIRECTORY ${CMAKE_BINARY_DIR}/resources DESTINATION .)
install(DIRECTORY ${CMAKE_BINARY_DIR}/output DESTINATION .)