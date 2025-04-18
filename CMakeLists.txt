cmake_minimum_required(VERSION 3.14)
project(lua_debug_wrapper)

include(FetchContent)

###############################################################################
# Fetch and Build "luamake"
###############################################################################

FetchContent_Declare(
  luamake
  GIT_REPOSITORY https://github.com/actboy168/luamake.git
  GIT_TAG        master  # Pin to a specific commit/branch if desired.
)
FetchContent_MakeAvailable(luamake)

add_custom_target(build_luamake
  COMMAND ${CMAKE_COMMAND} -E echo "Building luamake..."
  COMMAND ${CMAKE_COMMAND} -E chdir ${luamake_SOURCE_DIR} bash compile/build.sh
  WORKING_DIRECTORY ${luamake_SOURCE_DIR}
  COMMENT "Compiling luamake"
  VERBATIM
)

set(LUAMAKE_EXECUTABLE "${luamake_SOURCE_DIR}/luamake")

###############################################################################
# Fetch and Build "lua-debug"
###############################################################################

FetchContent_Declare(
  lua_debug
  GIT_REPOSITORY https://github.com/actboy168/lua-debug.git
  GIT_TAG        master
)
FetchContent_MakeAvailable(lua_debug)

add_custom_target(build_lua_debug
  COMMAND ${CMAKE_COMMAND} -E echo "Initializing lua-debug submodules..."
  COMMAND ${CMAKE_COMMAND} -E chdir ${lua_debug_SOURCE_DIR} git submodule init
  COMMAND ${CMAKE_COMMAND} -E chdir ${lua_debug_SOURCE_DIR} git submodule update
  COMMAND ${CMAKE_COMMAND} -E echo "Downloading dependencies via luamake..."
  COMMAND ${LUAMAKE_EXECUTABLE} lua compile/download_deps.lua
  COMMAND ${CMAKE_COMMAND} -E echo "Building lua-debug in release mode..."
  COMMAND ${LUAMAKE_EXECUTABLE} -mode release
  WORKING_DIRECTORY ${lua_debug_SOURCE_DIR}
  DEPENDS build_luamake
  COMMENT "Building lua-debug with custom commands"
  VERBATIM
)

# Set LUADEBUG_ROOT to be a directory under the build folder.
set(LUADEBUG_ROOT "${CMAKE_BINARY_DIR}/luadebug")

###############################################################################
# Define Variables Based on LUADEBUG_ROOT
###############################################################################

# Define the path variables relative to LUADEBUG_ROOT.
set(LUADEBUG_51_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua51/luadebug.so")
set(LUADEBUG_52_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua52/luadebug.so")
set(LUADEBUG_53_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua53/luadebug.so")
set(LUADEBUG_54_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua54/luadebug.so")
set(LUADEBUG_JIT_SO "${LUADEBUG_ROOT}/runtime/linux-x64/luajit/luadebug.so")
set(LUADEBUG_SCRIPT "${LUADEBUG_ROOT}/script")

message(STATUS "LUADEBUG_ROOT: ${LUADEBUG_ROOT}")
message(STATUS "LUADEBUG_51_SO: ${LUADEBUG_51_SO}")
message(STATUS "LUADEBUG_52_SO: ${LUADEBUG_52_SO}")
message(STATUS "LUADEBUG_53_SO: ${LUADEBUG_53_SO}")
message(STATUS "LUADEBUG_54_SO: ${LUADEBUG_54_SO}")
message(STATUS "LUADEBUG_JIT_SO: ${LUADEBUG_JIT_SO}")
message(STATUS "LUADEBUG_SCRIPT: ${LUADEBUG_SCRIPT}")

###############################################################################
# Copy the "publish" Folder to the Build Directory
###############################################################################

# Create a custom target that copies the entire publish folder from lua-debug's source
# into ${LUADEBUG_ROOT}.
add_custom_target(copy_luadebug_publish ALL
  BYPRODUCTS
    ${LUADEBUG_51_SO}
    ${LUADEBUG_52_SO}
    ${LUADEBUG_53_SO}
    ${LUADEBUG_54_SO}
    ${LUADEBUG_JIT_SO}
  COMMAND ${CMAKE_COMMAND} -E remove_directory ${LUADEBUG_ROOT}
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${lua_debug_SOURCE_DIR}/publish ${LUADEBUG_ROOT}
  DEPENDS build_lua_debug
  COMMENT "Copying 'publish' folder from lua-debug to ${LUADEBUG_ROOT}"
)


###############################################################################
# Create Imported Targets for the Shared Libraries
###############################################################################

# Create an imported target for LuaJIT.
add_library(luadebug_jit SHARED IMPORTED GLOBAL)
set_target_properties(luadebug_jit PROPERTIES
  IMPORTED_LOCATION "${LUADEBUG_JIT_SO}"
  # Optionally, you can define other properties such as INTERFACE_INCLUDE_DIRECTORIES if needed.
)
# make sure the copy step runs before anyone tries to link luadebug_jit
add_dependencies(luadebug_jit copy_luadebug_publish)

# Similarly, if your consumers might need the other versions, you can create imported targets:
add_library(luadebug_51 SHARED IMPORTED GLOBAL)
set_target_properties(luadebug_51 PROPERTIES IMPORTED_LOCATION "${LUADEBUG_51_SO}")
add_dependencies(luadebug_51 copy_luadebug_publish)

add_library(luadebug_52 SHARED IMPORTED GLOBAL)
set_target_properties(luadebug_52 PROPERTIES IMPORTED_LOCATION "${LUADEBUG_52_SO}")
add_dependencies(luadebug_52 copy_luadebug_publish)

add_library(luadebug_53 SHARED IMPORTED GLOBAL)
set_target_properties(luadebug_53 PROPERTIES IMPORTED_LOCATION "${LUADEBUG_53_SO}")
add_dependencies(luadebug_53 copy_luadebug_publish)

add_library(luadebug_54 SHARED IMPORTED GLOBAL)
set_target_properties(luadebug_54 PROPERTIES IMPORTED_LOCATION "${LUADEBUG_54_SO}")
add_dependencies(luadebug_54 copy_luadebug_publish)

###############################################################################
# Overall Wrapper Target
###############################################################################

# This target ensures all steps are executed.
add_custom_target(all_lua_debug_build ALL
  DEPENDS build_luamake build_lua_debug copy_luadebug_publish
  COMMENT "Building all components for lua-debug wrapper"
)
