cmake_minimum_required(VERSION 3.14)
project(lua_debug_wrapper)

include(FetchContent)

###############################################################################
# Fetch and Build "luamake"
###############################################################################

# Download the luamake repository.
FetchContent_Declare(
  luamake
  GIT_REPOSITORY https://github.com/actboy168/luamake.git
  GIT_TAG        master  # Pin to a specific commit/branch if desired.
)
FetchContent_MakeAvailable(luamake)

# Build luamake using the provided script.
add_custom_target(build_luamake
  COMMAND ${CMAKE_COMMAND} -E echo "Building luamake..."
  COMMAND ${CMAKE_COMMAND} -E chdir ${luamake_SOURCE_DIR} bash compile/build.sh
  WORKING_DIRECTORY ${luamake_SOURCE_DIR}
  COMMENT "Compiling luamake"
  VERBATIM
)

# Define the location of the built luamake executable.
set(LUAMAKE_EXECUTABLE "${luamake_SOURCE_DIR}/luamake")


###############################################################################
# Fetch and Build "lua-debug"
###############################################################################

# Download the lua-debug repository.
FetchContent_Declare(
  lua_debug
  GIT_REPOSITORY https://github.com/actboy168/lua-debug.git
  GIT_TAG        master
)
FetchContent_MakeAvailable(lua_debug)

# Build lua-debug with its custom steps.
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

###############################################################################
# Copy the "publish" Folder to the Build Directory
###############################################################################

# Set LUADEBUG_ROOT to be a directory under the build folder.
set(LUADEBUG_ROOT "${CMAKE_BINARY_DIR}/luadebug")

# Create a custom target that copies the entire publish folder from the lua-debug
# source location into ${LUADEBUG_ROOT}. This prevents the output from remaining
# in the source tree.
add_custom_target(copy_luadebug_publish ALL
  COMMAND ${CMAKE_COMMAND} -E remove_directory ${LUADEBUG_ROOT}
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${lua_debug_SOURCE_DIR}/publish ${LUADEBUG_ROOT}
  DEPENDS build_lua_debug
  COMMENT "Copying 'publish' folder from lua-debug to ${LUADEBUG_ROOT}"
)

###############################################################################
# Set Variables Based on the New LUADEBUG_ROOT
###############################################################################

# Define a variable for each lua architecture version. These are computed
# relative to the new LUADEBUG_ROOT folder. Adjust the subfolder names if they
# differ from your actual build output.
set(LUADEBUG_51_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua51/luadebug.so")
set(LUADEBUG_52_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua52/luadebug.so")
set(LUADEBUG_53_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua53/luadebug.so")
set(LUADEBUG_54_SO  "${LUADEBUG_ROOT}/runtime/linux-x64/lua54/luadebug.so")
set(LUADEBUG_JIT_SO "${LUADEBUG_ROOT}/runtime/linux-x64/luajit/luadebug.so")

# Define LUADEBUG_SCRIPT as a subdirectory where additional lua scripts may reside.
set(LUADEBUG_SCRIPT "${LUADEBUG_ROOT}/script")

# Optionally, print the variables for confirmation.
message(STATUS "LUADEBUG_ROOT: ${LUADEBUG_ROOT}")
message(STATUS "LUADEBUG_51_SO: ${LUADEBUG_51_SO}")
message(STATUS "LUADEBUG_52_SO: ${LUADEBUG_52_SO}")
message(STATUS "LUADEBUG_53_SO: ${LUADEBUG_53_SO}")
message(STATUS "LUADEBUG_54_SO: ${LUADEBUG_54_SO}")
message(STATUS "LUADEBUG_JIT_SO: ${LUADEBUG_JIT_SO}")
message(STATUS "LUADEBUG_SCRIPT: ${LUADEBUG_SCRIPT}")

###############################################################################
# Overall Wrapper Target
###############################################################################

# Create a target that depends on building both dependencies and the copying step.
add_custom_target(all_lua_debug_build ALL
  DEPENDS build_luamake build_lua_debug copy_luadebug_publish
  COMMENT "Building all components for lua-debug wrapper"
)
