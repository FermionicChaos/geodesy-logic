cmake_minimum_required(VERSION 3.20)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(PROJECT_NAME "geodesy-logic")

# ----------------------- Dependencies ----------------------- #

include(FetchContent)

# Utility function for fetching dependencies in one line.
function(fetch_dependency ADEP_NAME AGIT_REPO AGIT_TAG)
    # Remaining args are variable=value pairs for cache options
    set(options)
    set(oneValueArgs)
    set(multiValueArgs)
    cmake_parse_arguments(FETCHDEP "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    # Declare the dependency
    FetchContent_Declare(
        ${ADEP_NAME}
        GIT_REPOSITORY ${AGIT_REPO}
        GIT_TAG        ${AGIT_TAG}
        SOURCE_DIR     ${CMAKE_SOURCE_DIR}/dep/${ADEP_NAME}
    )
    
    # Apply cache variables with intelligent type detection and FORCE override
    foreach(arg IN LISTS ARGN)
        string(REPLACE "=" ";" kv "${arg}")
        list(GET kv 0 key)
        list(GET kv 1 val)
       
        # Determine the appropriate cache variable type based on value
        if(val MATCHES "^(ON|OFF|TRUE|FALSE)$")
            # Boolean values
            set(${key} ${val} CACHE BOOL "Forced by fetch_dependency for ${ADEP_NAME}" FORCE)
        else()
            # Everything else as STRING
            set(${key} ${val} CACHE STRING "Forced by fetch_dependency for ${ADEP_NAME}" FORCE)
        endif()
    endforeach()
    
    # Actually fetch it - cache variables should now be available to subdirectory
    FetchContent_MakeAvailable(${ADEP_NAME})
endfunction()

# Fetch a dependency that does NOT provide CMakeLists.txt.
# Populates sources into ${CMAKE_SOURCE_DIR}/dep/<name> and exposes <name>_SOURCE_DIR.
function(fetch_source_dependency ADEP_NAME AGIT_REPO AGIT_TAG)
    FetchContent_Declare(
        ${ADEP_NAME}
        GIT_REPOSITORY ${AGIT_REPO}
        GIT_TAG        ${AGIT_TAG}
        SOURCE_DIR     ${CMAKE_SOURCE_DIR}/dep/${ADEP_NAME}
    )

    FetchContent_GetProperties(${ADEP_NAME})
    if(NOT ${ADEP_NAME}_POPULATED)
        FetchContent_Populate(${ADEP_NAME})
    endif()

    # FetchContent_* sets these in the current scope; export them for callers.
    set(${ADEP_NAME}_SOURCE_DIR "${${ADEP_NAME}_SOURCE_DIR}" PARENT_SCOPE)
    set(${ADEP_NAME}_BINARY_DIR "${${ADEP_NAME}_BINARY_DIR}" PARENT_SCOPE)
endfunction()

# ----------------------- Lua (C) ----------------------- #
# Lua upstream does not ship CMake; we fetch sources and build a static library.
# Pinned tag for reproducible builds.
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

file(GLOB_RECURSE INC
    "inc/*.h"
)

file(GLOB_RECURSE SRC
    "src/*.h"
    "src/*.cpp"
    "src/*.c"
)

project(${PROJECT_NAME})

add_library(${PROJECT_NAME} ${INC} ${SRC})

target_include_directories(${PROJECT_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/inc/)

# Make Lua available to consumers of geodesy-logic.
target_link_libraries(${PROJECT_NAME} PUBLIC lua::lua)
