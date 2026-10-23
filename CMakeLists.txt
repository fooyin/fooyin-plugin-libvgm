cmake_minimum_required(VERSION 3.19)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

project(
    vgminput
    VERSION 1.0.4
    DESCRIPTION "Libvgm plugin for fooyin"
    HOMEPAGE_URL "https://github.com/fooyin/fooyin-plugin-libvgm"
    LANGUAGES CXX
)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)

option(BUILD_PLUGIN_BUNDLE "Enable the package_plugin target using bundled libvgm" OFF)

find_package(Fooyin)
if(BUILD_PLUGIN_BUNDLE)
    set(LIBRARY_TYPE
        SHARED
        CACHE STRING "libvgm library build type"
    )
else()
    find_package(LibVgm QUIET MODULE)
endif()

if(NOT BUILD_PLUGIN_BUNDLE AND LIBVGM_FOUND)
    message(STATUS "Using system libvgm")
elseif(EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/libvgm/CMakeLists.txt")
    message(STATUS "Using 3rd-party libvgm")

    if(WIN32)
        # Bundled libvgm ships a 32-bit iconv library; use native Windows conversion instead
        set(UTIL_CHARCNV_ICONV
            OFF
            CACHE BOOL "Use iconv for libvgm charset conversion"
        )
        set(UTIL_CHARCNV_WINAPI
            ON
            CACHE BOOL "Use WinAPI for libvgm charset conversion"
        )
    endif()

    add_subdirectory(${CMAKE_SOURCE_DIR}/3rdparty/libvgm libvgm EXCLUDE_FROM_ALL)
    target_link_libraries(vgm-player INTERFACE vgm-utils)
    add_library(libvgm::vgm-player ALIAS vgm-player)
else()
    if(BUILD_PLUGIN_BUNDLE)
        message(
            FATAL_ERROR
                "Plugin bundles require the libvgm submodule; initialise it with git submodule update --init --recursive"
        )
    endif()
    message(STATUS "libvgm not found; skipping libvgm plugin")
    return()
endif()

create_fooyin_plugin(
    vgminput
    JSON_IN src/vgminput.json.in
    DEPENDS Fooyin::Core
            Fooyin::Gui
            libvgm::vgm-player
    SOURCES src/vgminput.h
            src/vgminput.cpp
            src/vgminputplugin.cpp
            src/vgminputplugin.h
            src/vgminputsettings.cpp
            src/vgminputsettings.h
)

if(BUILD_PLUGIN_BUNDLE)
    set(bundle_runtime_files)
    set(bundle_license_files)
    if(WIN32)
        foreach(bundle_target vgminput vgm-player vgm-emu vgm-utils)
            set_target_properties(${bundle_target} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)
        endforeach()

        get_filename_component(zlib_library_dir "${ZLIB_LIBRARY_RELEASE}" DIRECTORY)
        find_file(
            PLUGIN_BUNDLE_ZLIB_RUNTIME
            NAMES zlib1.dll
            HINTS "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/bin" "${zlib_library_dir}/../bin"
        )
        find_file(
            PLUGIN_BUNDLE_ZLIB_LICENSE
            NAMES copyright
            HINTS "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/share/zlib" "${zlib_library_dir}/../share/zlib"
        )
        if(NOT PLUGIN_BUNDLE_ZLIB_RUNTIME OR NOT PLUGIN_BUNDLE_ZLIB_LICENSE)
            message(
                FATAL_ERROR
                    "Set PLUGIN_BUNDLE_ZLIB_RUNTIME and PLUGIN_BUNDLE_ZLIB_LICENSE to the zlib DLL and licence file"
            )
        endif()
        list(APPEND bundle_runtime_files "${PLUGIN_BUNDLE_ZLIB_RUNTIME}")
        set(bundle_license_files "${CMAKE_CURRENT_BINARY_DIR}/bundle-licenses/zlib.txt")
        configure_file("${PLUGIN_BUNDLE_ZLIB_LICENSE}" "${bundle_license_files}" COPYONLY)
    endif()

    create_fooyin_plugin_bundle(
        vgminput
        NAME
        fooyin-libvgm
        RUNTIME_TARGETS
        vgm-player
        vgm-emu
        vgm-utils
        RUNTIME_FILES
        ${bundle_runtime_files}
        FILES
        COPYING
        README.md
        LICENSE_FILES
        ${bundle_license_files}
    )
endif()
