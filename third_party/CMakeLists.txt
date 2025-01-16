# Copyright (c) 2019-2024, Arm Limited and Contributors
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

cmake_minimum_required(VERSION 3.10)

project(third_party LANGUAGES C CXX)

add_library(custom_vulkan INTERFACE)

target_compile_definitions(custom_vulkan INTERFACE VK_NO_PROTOTYPES)

if(VKB_ENABLE_PORTABILITY)
    target_compile_definitions(custom_vulkan INTERFACE VK_ENABLE_BETA_EXTENSIONS)
endif()

if(ANDROID)
    target_compile_definitions(custom_vulkan INTERFACE VK_USE_PLATFORM_ANDROID_KHR)
elseif(WIN32)
    target_compile_definitions(custom_vulkan INTERFACE VK_USE_PLATFORM_WIN32_KHR)
elseif(APPLE)
    target_compile_definitions(custom_vulkan INTERFACE VK_USE_PLATFORM_METAL_EXT)
elseif(UNIX)
    if (VKB_WSI_SELECTION STREQUAL XCB)
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(XCB xcb REQUIRED)
        if (XCB_FOUND)
            target_compile_definitions(custom_vulkan INTERFACE VK_USE_PLATFORM_XCB_KHR)
        endif()
    elseif (VKB_WSI_SELECTION STREQUAL XLIB)
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(X11 x11 REQUIRED)
        if (X11_FOUND)
            target_compile_definitions(custom_vulkan INTERFACE VK_USE_PLATFORM_XLIB_KHR)
        endif()
    elseif (VKB_WSI_SELECTION STREQUAL WAYLAND)
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(WAYLAND wayland-client REQUIRED)
        if (WAYLAND_FOUND)
            target_compile_definitions(custom_vulkan INTERFACE VK_USE_PLATFORM_WAYLAND_KHR)
        endif()
    elseif (VKB_WSI_SELECTION STREQUAL D2D)
        set(DIRECT_TO_DISPLAY TRUE)
        set(DIRECT_TO_DISPLAY TRUE PARENT_SCOPE)
        target_compile_definitions(custom_vulkan INTERFACE VK_USE_PLATFORM_DISPLAY_KHR)
    else()
        message(FATAL_ERROR "Unknown WSI")
    endif()
endif()


find_package(VulkanMemoryAllocator CONFIG REQUIRED)

target_link_libraries(custom_vulkan INTERFACE Vulkan::Vulkan GPUOpen::VulkanMemoryAllocator)

if(ANDROID)
# hwcpipe
add_subdirectory(hwcpipe)
set_target_properties(hwcpipe PROPERTIES FOLDER "ThirdParty" POSITION_INDEPENDENT_CODE ON)
endif()

# Allow the user to specify the ASTC architecture explicitly if building for a platform
# other than the host architecture (i.e. x86 on an x64 CPU)
if (NOT (DEFINED ASTC_ARCH))
    # On an x86_64 macbook, AVX2 is selected, but linking fails with this message,
    # `ld: object file .../libastcdec-avx2-static.a was built for different x86_64 sub-type (8) than link command line (3)`
    # so we fallback to native. If we're on an ARM based mac, there's no need for any SIMD extension, because they support
    # ASTC_LDR natively, so this library won't even be used.
    if (APPLE)
        set(ASTC_ARCH NATIVE)
    else()
        include( FindAVX2 )
        include( FindSSE2 )
        include( FindNEON )
        if (${AVX2_FOUND})
            set(ASTC_ARCH AVX2)
        elseif(${SSE2_FOUND})
            set(ASTC_ARCH SSE2)
        elseif(${NEON_FOUND})
            set(ASTC_ARCH NEON)
        else()
            set(ASTC_ARCH NATIVE)
        endif()
    endif()
endif()

string(TOUPPER ${ASTC_ARCH} ASTC_ARCH)
string(TOLOWER ${ASTC_ARCH} ASTC_ARCH_LOWER)

set(ASTCENC_ISA_${ASTC_ARCH} ON)
set(ASTCENC_CLI OFF)
set(ASTCENC_UNITTEST OFF)
set(ASTCENC_DECOMPRESSOR ON)
set(ASTCENC_UNIVERSAL_BUILD OFF)
set(ASTC_RAW_TARGET astcdec-${ASTC_ARCH_LOWER}-static)
set(ASTC_TARGET ${ASTC_RAW_TARGET} PARENT_SCOPE)

add_subdirectory(astc)

# ASTC apparently tries to build for x86_64 even on Mac arm64 architectures,
# but we can force it to build for the correct arch
# Upstream bug: https://github.com/ARM-software/astc-encoder/issues/458
if(IOS)
	if(${CMAKE_OSX_SYSROOT} STREQUAL "iphonesimulator")
		# handle iOS Simulator case, which should always match CMAKE_HOST_SYSTEM_PROCESSOR
		set(CMAKE_SYSTEM_PROCESSOR ${CMAKE_HOST_SYSTEM_PROCESSOR})
	else()
		set(CMAKE_SYSTEM_PROCESSOR arm64)
	endif()
endif ()
if (APPLE AND (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "arm64"))
    set_target_properties(${ASTC_RAW_TARGET} PROPERTIES OSX_ARCHITECTURES "arm64")
endif()

# astc doesn't have separate directories for it's source code and public interface.  Additionally, it includes it's
# own copy of STB. In order to avoid conflicts, we copy the only header we need to the build directory and alter the
# INTERFACE_INCLUDE_DIRECTORIES of the target
if (NOT TARGET ${ASTC_RAW_TARGET})
    message(FATAL_ERROR "Couldn't find expected ASTC library target")
endif()

get_target_property(ASTC_BINARY_DIR ${ASTC_RAW_TARGET} BINARY_DIR)
file(
    COPY astc/Source/astcenc.h
    DESTINATION ${ASTC_BINARY_DIR}/Include)

    set_target_properties(${ASTC_RAW_TARGET} PROPERTIES
    INTERFACE_INCLUDE_DIRECTORIES ""
    INCLUDE_DIRECTORIES ""
    INTERFACE_SYSTEM_INCLUDE_DIRECTORIES ${ASTC_BINARY_DIR}/Include
)
target_include_directories(${ASTC_RAW_TARGET} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/astc/Source)
target_include_directories(${ASTC_RAW_TARGET} SYSTEM INTERFACE  ${ASTC_BINARY_DIR}/Include)

if (MSVC)
  target_compile_definitions(${ASTC_RAW_TARGET} PRIVATE _CRT_SECURE_NO_WARNINGS)
  if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(gtest PUBLIC -Wno-deprecated-declarations)
  endif ()
endif ()
set_property(TARGET ${ASTC_RAW_TARGET} PROPERTY FOLDER "ThirdParty")
