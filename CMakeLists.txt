cmake_minimum_required(VERSION 3.21)
set(PROJECT_NAME UAKTemplate)
project(${PROJECT_NAME})
set(CMAKE_CXX_STANDARD 20)
option(ULTRA_ENGINE_KIT "Switch Between GUI, 2D and 3D" GUI)

# Source: https://stackoverflow.com/a/7788165
MACRO(SUBDIRLIST result curdir)
    FILE(GLOB children RELATIVE ${curdir} ${curdir}/*)
    SET(dirlist "")
    FOREACH (child ${children})
        IF (IS_DIRECTORY ${curdir}/${child})
            LIST(APPEND dirlist ${child})
        ENDIF ()
    ENDFOREACH ()
    SET(${result} ${dirlist})
ENDMACRO()

include_directories(${UAK_ROOT_DIRECTORY}/Include)
# 3D and 2D are unusable as of now.
# The required include directories might change, depending on
# the changes made by Leadwerks in that regard.
if (ULTRA_ENGINE_KIT STREQUAL 3D)
    add_compile_definitions(${PROJECT_NAME} _ULTRA_ENGINE)
    include_directories(${UAK_ROOT_DIRECTORY}/Include/Libraries/Lua/src)
    include_directories(${UAK_ROOT_DIRECTORY}/Include/Libraries/sol3/include)

    # All Newton Dynamics Includes
    SUBDIRLIST(NEWTON_DYNAMICS_SDKS ${UAK_ROOT_DIRECTORY}/Include/Libraries/NewtonDynamics/sdk)
    FOREACH (NEWTON_DYNAMICS_SDK ${NEWTON_DYNAMICS_SDKS})
        if (NOT (NEWTON_DYNAMICS_SDK MATCHES "projects" OR NEWTON_DYNAMICS_SDK MATCHES "thirdParty"))
            include_directories(${UAK_ROOT_DIRECTORY}/Include/Libraries/NewtonDynamics/sdk/${NEWTON_DYNAMICS_SDK})
            message(${NEWTON_DYNAMICS_SDK})
        endif ()
    ENDFOREACH ()

    # All RecastNavigation Includes
    SUBDIRLIST(RECAST_NAVIGATIONS ${UAK_ROOT_DIRECTORY}/Include/Libraries/RecastNavigation)
    FOREACH (RECAST_NAVIGATION_INCLUDE ${RECAST_NAVIGATIONS})
        if (NOT (RECAST_NAVIGATION_INCLUDE MATCHES "Tests" OR RECAST_NAVIGATION_INCLUDE MATCHES "Docs" OR RECAST_NAVIGATION_INCLUDE MATCHES "RecastDemo"))
            include_directories(${UAK_ROOT_DIRECTORY}/Include/Libraries/RecastNavigation/${RECAST_NAVIGATION_INCLUDE}/Include)
            message(${RECAST_NAVIGATION_INCLUDE})
        endif ()
    ENDFOREACH ()
elseif (ULTRA_ENGINE_KIT STREQUAL 2D)
    add_compile_definitions(${PROJECT_NAME} _ULTRA_2DKIT)
    include_directories(${UAK_ROOT_DIRECTORY}/Include/Libraries/FreeType/include)
    include_directories(${UAK_ROOT_DIRECTORY}/Include/Libraries/Box2D)
else ()
    add_compile_definitions(${PROJECT_NAME} _ULTRA_APPKIT)
endif ()

# Platform and Architecture specific checks
if (CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(ARCH "x64")
elseif (CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(ARCH "x86")
else ()
    set(ARCH "Unknown")
endif ()

if (${CMAKE_SYSTEM_NAME} STREQUAL "Windows")

    if (ARCH STREQUAL "Unknown")
        message(FATAL_ERROR Unknown Windows arch.)
    endif ()

    set(UAK_LIB_FILE AppKit.lib)
    set(OS_NAME Windows)

elseif (${CMAKE_SYSTEM_NAME} STREQUAL "Darwin")

    execute_process(COMMAND /usr/bin/uname -m OUTPUT_VARIABLE SYSTEM_MACOS_ARCH OUTPUT_STRIP_TRAILING_WHITESPACE)

    if (NOT DEFINED ARCH)
        set(MACOS_ARCH "${SYSTEM_MACOS_ARCH}")
    endif ()

    if ("${MACOS_ARCH}" MATCHES "^x86_64")
        set(ARCH "x64")
    elseif ("${MACOS_ARCH}" MATCHES "^arm64")
        set(ARCH "arm64")
    else ()
        message(FATAL_ERROR Unknown MacOS arch.)
    endif ()

    set(UAK_LIB_FILE libAppKit.a)
    set(OS_NAME MacOS)

elseif (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")

    if (ARCH NOT STREQUAL "x64")
        message(FATAL_ERROR Unsupported Linux arch.)
    endif ()

    set(UAK_LIB_FILE AppKit.o)
    set(OS_NAME Linux)

endif ()

# Final location of the library file to be linked against.
set(UAK_LIBRARY "${UAK_ROOT_DIRECTORY}/Library/${OS_NAME}/${ARCH}/${CMAKE_BUILD_TYPE}/${UAK_LIB_FILE}")

if (${CMAKE_BUILD_TYPE} MATCHES Release)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup")
    add_definitions(-MT)
elseif (${CMAKE_BUILD_TYPE} MATCHES Debug)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SUBSYSTEM:CONSOLE /ENTRY:mainCRTStartup")
    add_definitions(-MTd)
    add_compile_definitions(${PROJECT_NAME} _DEBUG)
endif ()


# Your personal stuff
add_executable(${PROJECT_NAME} src/main.cpp)

target_link_libraries(${PROJECT_NAME} ${UAK_LIBRARY})


# UltraEngine specific stuff
# Precompile Header for UltraEngine Header
target_precompile_headers(${PROJECT_NAME} PRIVATE [["UltraEngine.h"]])