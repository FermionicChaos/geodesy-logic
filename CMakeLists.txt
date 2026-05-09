cmake_minimum_required(VERSION 3.20)

# Repository metadata
set(REPO_NAME "geodesy-logic")
set(REPO_VERSION_MAJOR 0)
set(REPO_VERSION_MINOR 0)
set(REPO_VERSION_PATCH 0)
set(REPO_BRANCH "unknown")
set(REPO_UNIT_TEST_NAME "${REPO_NAME}-unit-test")

# ==============================================================================
# Git Metadata
# ==============================================================================

find_package(Git QUIET)

if(GIT_FOUND AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git")
    execute_process(
        COMMAND "${GIT_EXECUTABLE}" rev-parse --abbrev-ref HEAD
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        OUTPUT_VARIABLE REPO_BRANCH_RESULT
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )

    if(NOT REPO_BRANCH_RESULT STREQUAL ""
       AND NOT REPO_BRANCH_RESULT STREQUAL "HEAD")
        set(REPO_BRANCH "${REPO_BRANCH_RESULT}")
    endif()

    execute_process(
        COMMAND "${GIT_EXECUTABLE}" describe --exact-match --tags HEAD
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        OUTPUT_VARIABLE REPO_GIT_TAG
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
        RESULT_VARIABLE REPO_GIT_TAG_RESULT
    )

    if(REPO_GIT_TAG_RESULT EQUAL 0)
        string(REGEX REPLACE "^v" "" REPO_GIT_TAG "${REPO_GIT_TAG}")
        if(REPO_GIT_TAG MATCHES "^([0-9]+)\\.([0-9]+)\\.([0-9]+)$")
            set(REPO_VERSION_MAJOR "${CMAKE_MATCH_1}")
            set(REPO_VERSION_MINOR "${CMAKE_MATCH_2}")
            set(REPO_VERSION_PATCH "${CMAKE_MATCH_3}")
        endif()
    endif()
endif()

# Set New Version String for use in code and packaging.
set(REPO_VERSION "${REPO_VERSION_MAJOR}.${REPO_VERSION_MINOR}.${REPO_VERSION_PATCH}")

# Include all public headers.
file(GLOB_RECURSE REPO_INC CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/inc/*.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/inc/*.hh"
    "${CMAKE_CURRENT_SOURCE_DIR}/inc/*.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/inc/*.hxx"
)

# Include all source files and private headers (if any).
file(GLOB_RECURSE REPO_SRC CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cc"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cxx"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.hh"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.hxx"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.inl"
)

set(REPO_DEPENDENCY_DIR "${CMAKE_SOURCE_DIR}/dep" CACHE PATH "Select Path to store fetched dependency source code. Default is ${CMAKE_SOURCE_DIR}/dep/")

# Declare main project
project(${REPO_NAME}-development
    VERSION ${REPO_VERSION}
    LANGUAGES CXX
)

# Set Project Global Settings
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

# Utility function for fetching dependencies in one line.
function(fetch_dependency ADEP_NAME AGIT_REPO AGIT_TAG)
    FetchContent_Declare(
        ${ADEP_NAME}
        GIT_REPOSITORY "${AGIT_REPO}"
        GIT_TAG        "${AGIT_TAG}"
        SOURCE_DIR     "${REPO_DEPENDENCY_DIR}/${ADEP_NAME}"
    )

    # Apply cache variables with intelligent type detection and FORCE override
    # CRITICAL: Set these BEFORE FetchContent_MakeAvailable
    foreach(arg IN LISTS ARGN)
        if(arg MATCHES "^([^=]+)=(.*)$")
            set(key "${CMAKE_MATCH_1}")
            set(val "${CMAKE_MATCH_2}")
        else()
            message(FATAL_ERROR
                "Invalid argument passed to fetch_dependency(${ADEP_NAME}): '${arg}'. Expected KEY=VALUE."
            )
        endif()

        if(val MATCHES "^(ON|OFF|TRUE|FALSE)$")
            set(${key} ${val} CACHE BOOL "Forced by fetch_dependency for ${ADEP_NAME}" FORCE)
        else()
            set(${key} "${val}" CACHE STRING "Forced by fetch_dependency for ${ADEP_NAME}" FORCE)
        endif()
    endforeach()

    FetchContent_MakeAvailable(${ADEP_NAME})
endfunction()

# Fetch a dependency that does NOT provide CMakeLists.txt.
# Populates sources into ${REPO_DEPENDENCY_DIR}/<name> and exposes <name>_SOURCE_DIR.
function(fetch_source_dependency ADEP_NAME AGIT_REPO AGIT_TAG)
    FetchContent_Declare(
        ${ADEP_NAME}
        GIT_REPOSITORY "${AGIT_REPO}"
        GIT_TAG        "${AGIT_TAG}"
        SOURCE_DIR     "${REPO_DEPENDENCY_DIR}/${ADEP_NAME}"
    )

    FetchContent_GetProperties(${ADEP_NAME})
    if(NOT ${ADEP_NAME}_POPULATED)
        FetchContent_Populate(${ADEP_NAME})
    endif()

    set(${ADEP_NAME}_SOURCE_DIR "${${ADEP_NAME}_SOURCE_DIR}" PARENT_SCOPE)
    set(${ADEP_NAME}_BINARY_DIR "${${ADEP_NAME}_BINARY_DIR}" PARENT_SCOPE)
endfunction()

# ==============================================================================
# Lua (C) — upstream does not ship CMake; build a static library from sources.
# ==============================================================================

fetch_source_dependency(lua https://github.com/lua/lua.git v5.4.7)

set(LUA_SRC_DIR "${lua_SOURCE_DIR}/src")
if(NOT EXISTS "${LUA_SRC_DIR}/lapi.c")
    # The official lua/lua git repo keeps sources at repo root.
    set(LUA_SRC_DIR "${lua_SOURCE_DIR}")
endif()

add_library(lua_c STATIC
    "${LUA_SRC_DIR}/lapi.c"
    "${LUA_SRC_DIR}/lauxlib.c"
    "${LUA_SRC_DIR}/lbaselib.c"
    "${LUA_SRC_DIR}/lcode.c"
    "${LUA_SRC_DIR}/lcorolib.c"
    "${LUA_SRC_DIR}/lctype.c"
    "${LUA_SRC_DIR}/ldblib.c"
    "${LUA_SRC_DIR}/ldebug.c"
    "${LUA_SRC_DIR}/ldo.c"
    "${LUA_SRC_DIR}/ldump.c"
    "${LUA_SRC_DIR}/lfunc.c"
    "${LUA_SRC_DIR}/lgc.c"
    "${LUA_SRC_DIR}/linit.c"
    "${LUA_SRC_DIR}/liolib.c"
    "${LUA_SRC_DIR}/llex.c"
    "${LUA_SRC_DIR}/lmathlib.c"
    "${LUA_SRC_DIR}/lmem.c"
    "${LUA_SRC_DIR}/loadlib.c"
    "${LUA_SRC_DIR}/lobject.c"
    "${LUA_SRC_DIR}/lopcodes.c"
    "${LUA_SRC_DIR}/loslib.c"
    "${LUA_SRC_DIR}/lparser.c"
    "${LUA_SRC_DIR}/lstate.c"
    "${LUA_SRC_DIR}/lstring.c"
    "${LUA_SRC_DIR}/lstrlib.c"
    "${LUA_SRC_DIR}/ltable.c"
    "${LUA_SRC_DIR}/ltablib.c"
    "${LUA_SRC_DIR}/ltm.c"
    "${LUA_SRC_DIR}/lundump.c"
    "${LUA_SRC_DIR}/lutf8lib.c"
    "${LUA_SRC_DIR}/lvm.c"
    "${LUA_SRC_DIR}/lzio.c"
)

add_library(lua::lua ALIAS lua_c)

target_include_directories(lua_c PUBLIC "${LUA_SRC_DIR}")
set_target_properties(lua_c PROPERTIES POSITION_INDEPENDENT_CODE ON)

if(UNIX AND NOT APPLE)
    target_compile_definitions(lua_c PRIVATE LUA_USE_LINUX)
    target_link_libraries(lua_c PRIVATE m dl)
endif()

add_library(${REPO_NAME} ${REPO_INC} ${REPO_SRC})

target_compile_definitions(${REPO_NAME} PUBLIC
    GEODESY_LOGIC_NAME="${REPO_NAME}"
    GEODESY_LOGIC_BRANCH="${REPO_BRANCH}"
    GEODESY_LOGIC_VERSION="${REPO_VERSION}"
    GEODESY_LOGIC_VERSION_MAJOR=${REPO_VERSION_MAJOR}
    GEODESY_LOGIC_VERSION_MINOR=${REPO_VERSION_MINOR}
    GEODESY_LOGIC_VERSION_PATCH=${REPO_VERSION_PATCH}
)

target_include_directories(${REPO_NAME}
    PUBLIC
        "${CMAKE_CURRENT_SOURCE_DIR}/inc"
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/src"
)

target_link_libraries(${REPO_NAME}
    PUBLIC
        lua::lua
)
