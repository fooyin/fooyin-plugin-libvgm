cmake_minimum_required(VERSION 3.14)

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
    set(LIBRARY_TYPE SHARED CACHE STRING "libvgm library build type")
else()
    find_package(LibVgm QUIET MODULE)
endif()

if(NOT BUILD_PLUGIN_BUNDLE AND LIBVGM_FOUND)
    message(STATUS "Using system libvgm")
elseif(EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/libvgm/CMakeLists.txt")
    message(STATUS "Using 3rd-party libvgm")

    if(WIN32)
        # Bundled libvgm ships a 32-bit iconv library; use native Windows conversion instead
        set(UTIL_CHARCNV_ICONV OFF CACHE BOOL "Use iconv for libvgm charset conversion")
        set(UTIL_CHARCNV_WINAPI ON CACHE BOOL "Use WinAPI for libvgm charset conversion")
    endif()

    add_subdirectory(${CMAKE_SOURCE_DIR}/3rdparty/libvgm libvgm EXCLUDE_FROM_ALL)
    target_link_libraries(vgm-player INTERFACE vgm-utils)
    add_library(libvgm::vgm-player ALIAS vgm-player)
else()
    if(BUILD_PLUGIN_BUNDLE)
        message(FATAL_ERROR "Plugin bundles require the libvgm submodule; initialise it with git submodule update --init --recursive")
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
    if(WIN32)
        set(bundle_os windows)
    elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        set(bundle_os linux)
    else()
        message(FATAL_ERROR "Plugin bundles are currently supported on Linux and Windows")
    endif()

    string(TOLOWER "${CMAKE_SYSTEM_PROCESSOR}" bundle_arch)
    if(bundle_arch MATCHES "^(amd64|x86_64)$" AND CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(bundle_arch x64)
    elseif(bundle_arch MATCHES "^(aarch64|arm64)$" AND CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(bundle_arch arm64)
    else()
        message(FATAL_ERROR "Unsupported plugin bundle architecture: ${CMAKE_SYSTEM_PROCESSOR}")
    endif()
    set(bundle_platform "${bundle_os}-${bundle_arch}")
    set(bundle_files "$<TARGET_FILE:vgminput>")

    foreach(bundle_target vgminput vgm-player vgm-emu vgm-utils)
        if(WIN32)
            set_target_properties(${bundle_target} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)
        else()
            set_property(TARGET ${bundle_target} APPEND PROPERTY BUILD_RPATH "$ORIGIN")
            set_target_properties(${bundle_target} PROPERTIES BUILD_RPATH_USE_ORIGIN ON)
        endif()
        get_target_property(bundle_target_type ${bundle_target} TYPE)
        if(bundle_target_type STREQUAL "SHARED_LIBRARY")
            list(APPEND bundle_files "$<TARGET_FILE:${bundle_target}>")
        endif()
    endforeach()

    if(WIN32)
        get_filename_component(zlib_library_dir "${ZLIB_LIBRARY_RELEASE}" DIRECTORY)
        find_file(
            PLUGIN_BUNDLE_ZLIB_RUNTIME
            NAMES zlib1.dll
            HINTS "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/bin"
                  "${zlib_library_dir}/../bin"
        )
        find_file(
            PLUGIN_BUNDLE_ZLIB_LICENSE
            NAMES copyright
            HINTS "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/share/zlib"
                  "${zlib_library_dir}/../share/zlib"
        )
        if(NOT PLUGIN_BUNDLE_ZLIB_RUNTIME OR NOT PLUGIN_BUNDLE_ZLIB_LICENSE)
            message(FATAL_ERROR "Set PLUGIN_BUNDLE_ZLIB_RUNTIME and PLUGIN_BUNDLE_ZLIB_LICENSE to the zlib DLL and licence file")
        endif()
        list(APPEND bundle_files "${PLUGIN_BUNDLE_ZLIB_RUNTIME}")
    endif()

    configure_file(cmake/PackagePlugin.cmake.in PackagePlugin.cmake.in @ONLY)
    file(
        GENERATE
        OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/PackagePlugin-$<CONFIG>.cmake"
        INPUT "${CMAKE_CURRENT_BINARY_DIR}/PackagePlugin.cmake.in"
    )
    add_custom_target(
        package_plugin
        COMMAND "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/PackagePlugin-$<CONFIG>.cmake"
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/fooyin-libvgm-${bundle_platform}.fyplugin"
        COMMENT "Packaging fooyin-libvgm-${bundle_platform}.fyplugin"
        VERBATIM
    )
    add_dependencies(package_plugin vgminput)
endif()
