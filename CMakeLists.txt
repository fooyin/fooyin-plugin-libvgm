cmake_minimum_required(VERSION 3.14)

project(
    vgminput
    VERSION 0.1.0
    DESCRIPTION "Libvgm plugin for fooyin"
    HOMEPAGE_URL "https://github.com/fooyin/fooyin-plugin-libvgm"
    LANGUAGES CXX
)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)

find_package(Fooyin)
find_package(LibVgm QUIET MODULE)

if(LIBVGM_FOUND)
    message(STATUS "Using system libvgm")
elseif(EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/libvgm/CMakeLists.txt")
    message(STATUS "Using 3rd-party libvgm")
    add_subdirectory(${CMAKE_SOURCE_DIR}/3rdparty/libvgm libvgm EXCLUDE_FROM_ALL)
    add_library(libvgm::vgm-player ALIAS vgm-player)
else()
    message(STATUS "libvgm not found; skipping libvgm plugin")
    return()
endif()

create_fooyin_plugin(
    vgminput
    JSON_IN src/vgminput.json.in
    DEPENDS Fooyin::Core
            libvgm::vgm-player
    SOURCES src/vgminput.h
            src/vgminput.cpp
            src/vgminputplugin.cpp
            src/vgminputplugin.h
            src/vgminputsettings.cpp
            src/vgminputsettings.h
)
