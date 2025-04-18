cmake_minimum_required(VERSION 3.14)
project(lua_debug_wrapper)

include(FetchContent)

###############################################################################
# 1) Fetch & Build luamake
###############################################################################
FetchContent_Declare(
  luamake
  GIT_REPOSITORY https://github.com/actboy168/luamake.git
  GIT_TAG        master
)
FetchContent_MakeAvailable(luamake)

add_custom_target(build_luamake
  COMMAND ${CMAKE_COMMAND} -E echo "Building luamake…"
  COMMAND ${CMAKE_COMMAND} -E chdir ${luamake_SOURCE_DIR} bash compile/build.sh
  DEPENDS ${luamake_SOURCE_DIR}
  COMMENT "Compiling luamake"
  VERBATIM
)

set(LUAMAKE_EXECUTABLE "${luamake_SOURCE_DIR}/luamake")


###############################################################################
# 2) Fetch & Build lua-debug
###############################################################################
FetchContent_Declare(
  lua_debug
  GIT_REPOSITORY https://github.com/actboy168/lua-debug.git
  GIT_TAG        master
)
FetchContent_MakeAvailable(lua_debug)

add_custom_target(build_lua_debug
  COMMAND ${CMAKE_COMMAND} -E echo "Initializing lua-debug submodules…"
  COMMAND ${CMAKE_COMMAND} -E chdir ${lua_debug_SOURCE_DIR} git submodule init
  COMMAND ${CMAKE_COMMAND} -E chdir ${lua_debug_SOURCE_DIR} git submodule update
  COMMAND ${CMAKE_COMMAND} -E echo "Downloading deps via luamake…"
  COMMAND ${LUAMAKE_EXECUTABLE} lua compile/download_deps.lua
  COMMAND ${CMAKE_COMMAND} -E echo "Building lua-debug (release)…"
  COMMAND ${LUAMAKE_EXECUTABLE} -mode release
  DEPENDS build_luamake
  COMMENT "Building lua-debug"
  VERBATIM
)


###############################################################################
# 3) Define LUADEBUG_ROOT & Paths for Each .so
###############################################################################
set(LUADEBUG_ROOT "${CMAKE_BINARY_DIR}/luadebug")

set(LUADEBUG_51_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua51/luadebug.so")
set(LUADEBUG_52_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua52/luadebug.so")
set(LUADEBUG_53_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua53/luadebug.so")
set(LUADEBUG_54_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua54/luadebug.so")
set(LUADEBUG_JIT_SO "${LUADEBUG_ROOT}/runtime/linux-x64/luajit/luadebug.so")

set(LUADEBUG_SCRIPT  "${LUADEBUG_ROOT}/script")


###############################################################################
# 4) Copy-publish Target (declares the .so as BYPRODUCTS)
###############################################################################
add_custom_target(copy_luadebug_publish
  BYPRODUCTS
    ${LUADEBUG_51_SO}
    ${LUADEBUG_52_SO}
    ${LUADEBUG_53_SO}
    ${LUADEBUG_54_SO}
    ${LUADEBUG_JIT_SO}
  COMMAND ${CMAKE_COMMAND} -E remove_directory "${LUADEBUG_ROOT}"
  COMMAND ${CMAKE_COMMAND} -E copy_directory
          "${lua_debug_SOURCE_DIR}/publish" "${LUADEBUG_ROOT}"
  DEPENDS build_lua_debug
  COMMENT "Copying lua-debug/publish → ${LUADEBUG_ROOT}"
  VERBATIM
)


###############################################################################
# 5) Create Imported Targets for Each Shared Library
###############################################################################
function(_create_luadebug_target name so_var)
  add_library(${name} SHARED IMPORTED GLOBAL)
  set_target_properties(${name} PROPERTIES
    IMPORTED_LOCATION "${${so_var}}"
  )
  # Ensure the .so is in place before linking:
  add_dependencies(${name} copy_luadebug_publish)
endfunction()

_create_luadebug_target(luadebug_51  LUADEBUG_51_SO)
_create_luadebug_target(luadebug_52  LUADEBUG_52_SO)
_create_luadebug_target(luadebug_53  LUADEBUG_53_SO)
_create_luadebug_target(luadebug_54  LUADEBUG_54_SO)
_create_luadebug_target(luadebug_jit LUADEBUG_JIT_SO)


###############################################################################
# 6) Optional: A Single “All” Target
###############################################################################
add_custom_target(all_lua_debug_build
  DEPENDS build_luamake build_lua_debug copy_luadebug_publish
  COMMENT "Building all lua-debug assets"
)
