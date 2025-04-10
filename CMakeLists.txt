cmake_minimum_required(VERSION 3.14)
project(lua_debug_wrapper)

include(FetchContent)

###############################################################################
# Fetch and build "luamake"
###############################################################################

# FetchContent makes the repository available in our build.
FetchContent_Declare(
  luamake
  GIT_REPOSITORY https://github.com/actboy168/luamake.git
  GIT_TAG        master  # You can fix to a commit, branch, or tag as needed.
)
FetchContent_MakeAvailable(luamake)

# Create a custom target to run the build steps for luamake.
# Note: We assume a Unix-like shell environment; for Windows you may need to adjust.
add_custom_target(build_luamake
  COMMAND ${CMAKE_COMMAND} -E echo "Building luamake..."
  COMMAND ${CMAKE_COMMAND} -E chdir ${luamake_SOURCE_DIR} bash compile/build.sh
  WORKING_DIRECTORY ${luamake_SOURCE_DIR}
  COMMENT "Compiling luamake"
  VERBATIM
)

# Define where the built executable is located (relative to luamake repository).
# This executable is not standalone—it depends on files in the repository.
set(LUAMAKE_EXECUTABLE "${luamake_SOURCE_DIR}/luamake")

###############################################################################
# Fetch and build "lua-debug"
###############################################################################

FetchContent_Declare(
  lua_debug
  GIT_REPOSITORY https://github.com/actboy168/lua-debug.git
  GIT_TAG        master  # Again, pin to a specific commit or tag if needed.
)
FetchContent_MakeAvailable(lua_debug)

# Create a custom target to build lua-debug.
# Note that we run the submodule init/update commands first and then use the built
# luamake to run the lua-debug-specific scripts.
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
# Expose the built artifact
###############################################################################

# The expected output binary is at:
# ${lua_debug_SOURCE_DIR}/publish/runtime/linux-x64/luajit/luadebug.so
# You can choose to install or copy this file to a known location.
set(LUADEBUG_OUTPUT "${lua_debug_SOURCE_DIR}/publish/runtime/linux-x64/luajit/luadebug.so")

# For example, let’s create an imported target so consuming projects can link to it:
add_library(luadebug SHARED IMPORTED GLOBAL)
set_target_properties(luadebug PROPERTIES
  IMPORTED_LOCATION "${LUADEBUG_OUTPUT}"
)

# Ensure the build of lua-debug happens before luadebug is used.
add_dependencies(luadebug build_lua_debug)

# Optionally, you can also add an install rule for the library:
install(FILES "${LUADEBUG_OUTPUT}" DESTINATION lib)

###############################################################################
# Wrapper target (optional)
###############################################################################
# You could define an overall target so that when consumers run "make all"
# both dependencies are built and the library becomes available.
add_custom_target(all_lua_debug_build ALL
  DEPENDS build_luamake build_lua_debug luadebug
  COMMENT "Building all components for lua-debug wrapper"
)
