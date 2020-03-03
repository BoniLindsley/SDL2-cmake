cmake_minimum_required(VERSION 3.0.2)

#
#
#
# ## Module dependencies

# Installation will create packages.
# The included module makes it easier to create them.
include(CMakePackageConfigHelpers)
# Installation will try use a standard file directory structure.
# The included module provides functions for that.
include(GNUInstallDirs)

#
#
#
# ## Defines a target for Make scripts, and used as export namespace.

project(SDL2)

#
#
#
# ## The SDL2 library wrapper

# Create a target that other CMake users can link to.
set(TARGET_NAME ${PROJECT_NAME})
add_library(${TARGET_NAME} INTERFACE)

# Tell CMake that the library built should be "installed" when asked to.
# In practise, since this is an interface library,
# this only creates a target, without installing anything.
install(TARGETS ${TARGET_NAME} EXPORT ${TARGET_NAME})

# Let other CMake users find this library after it is installed.
install(
  EXPORT ${TARGET_NAME}
  DESTINATION "${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake"
  NAMESPACE "${PROJECT_NAME}::"
  FILE "${TARGET_NAME}-target.cmake"
)

#
# ### Headers

find_path(
  SDL2_INCLUDE_DIR "SDL2/SDL.h"
  PATH_SUFFIXES "include"
  DOC "SDL2 include directory for '#include <SDL2/SDL.h>'"
)

find_path(
  SDL2_INCLUDE_SUBDIR "SDL.h"
  PATH_SUFFIXES "include" "include/SDL2"
  DOC "SDL2 include directory for '#include <SDL.h>'"
)

# In some distributions, `include/SDL2/SDL*.h` are provided.
# In that case `#include <SDL*.h>` is still supported
# because `include/SDL2` can be added to header search directory list.
#
# However, if `include/SDL*.h` is provided instead,
# then `#include <SDL2/SDL*.h>` is more difficult to support.
# This is done here by creating wrapper headers in `include/SDL2`
# that `#include`-s their corresponding headers in `include`.
if(SDL2_INCLUDE_SUBDIR AND NOT SDL2_INCLUDE_DIR)

  # Create a template from which the forwarding headers will be created.
  set(
    INCLUDE_TEMPLATE_DIR
    "${CMAKE_CURRENT_BINARY_DIR}/fake_include_template"
  )
  set(INCLUDE_TEMPLATE "${INCLUDE_TEMPLATE_DIR}/SDL_.h.in")
  file(
    WRITE "${INCLUDE_TEMPLATE}"
    [=[
#pragma once
#include <${HEADER_FILENAME}>
]=]
  )

  # Create a forwarding header for each required SDL2 header.
  list(
    APPEND
    HEADER_TO_FORWARD
    SDL.h
    SDL_assert.h
    SDL_atomic.h
    SDL_audio.h
    SDL_bits.h
    SDL_blendmode.h
    SDL_clipboard.h
    SDL_config.h
    SDL_cpuinfo.h
    SDL_egl.h
    SDL_endian.h
    SDL_error.h
    SDL_events.h
    SDL_filesystem.h
    SDL_gamecontroller.h
    SDL_gesture.h
    SDL_haptic.h
    SDL_hints.h
    SDL_joystick.h
    SDL_keyboard.h
    SDL_keycode.h
    SDL_loadso.h
    SDL_log.h
    SDL_main.h
    SDL_messagebox.h
    SDL_mouse.h
    SDL_mutex.h
    SDL_name.h
    SDL_opengl.h
    SDL_opengl_glext.h
    SDL_opengles.h
    SDL_opengles2.h
    SDL_opengles2_gl2.h
    SDL_opengles2_gl2ext.h
    SDL_opengles2_gl2platform.h
    SDL_opengles2_khrplatform.h
    SDL_pixels.h
    SDL_platform.h
    SDL_power.h
    SDL_quit.h
    SDL_rect.h
    SDL_render.h
    SDL_revision.h
    SDL_rwops.h
    SDL_scancode.h
    SDL_sensor.h
    SDL_shape.h
    SDL_stdinc.h
    SDL_surface.h
    SDL_system.h
    SDL_syswm.h
    SDL_test.h
    SDL_test_assert.h
    SDL_test_common.h
    SDL_test_compare.h
    SDL_test_crc32.h
    SDL_test_font.h
    SDL_test_fuzzer.h
    SDL_test_harness.h
    SDL_test_images.h
    SDL_test_log.h
    SDL_test_md5.h
    SDL_test_memory.h
    SDL_test_random.h
    SDL_thread.h
    SDL_timer.h
    SDL_touch.h
    SDL_version.h
    SDL_video.h
    SDL_vulkan.h
  )
  set(FAKE_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/fake_include")
  foreach(HEADER_FILENAME ${HEADER_TO_FORWARD})
    configure_file(
      "${INCLUDE_TEMPLATE}"
      "${FAKE_INCLUDE_DIR}/SDL2/${HEADER_FILENAME}"
    )
  endforeach()

  # The headers for `#inclue <SDL2/SDL*.h>` has now been "found".
  set(
    SDL2_INCLUDE_DIR
    "${FAKE_INCLUDE_DIR}"
    CACHE PATH DOC "SDL2 include directory"
  )

  # The headers will be installed so that other CMake users can find them
  # after the package is installed.
  install(DIRECTORY "${FAKE_INCLUDE_DIR}/SDL2" TYPE "INCLUDE")

endif()

# Let the user know there are problems finding installed headers.
# The user may not otherwise notice if they do not use the target.
if(NOT SDL2_INCLUDE_DIR AND NOT SDL2_INCLUDE_SUBDIR)
  message(SEND_ERROR "Unable to find header `SDL2/SDL.h` nor `SDL.h`.")
endif()

# Let user use the headers via, say, `add_subdirectory`.
target_include_directories(
  ${TARGET_NAME}
  INTERFACE $<BUILD_INTERFACE:${SDL2_INCLUDE_DIR}>
            $<BUILD_INTERFACE:${SDL2_INCLUDE_SUBDIR}>
)

#
# ### Link dependencies

find_library(
  SDL2_LIBRARY
  NAMES SDL2
  PATH_SUFFIXES lib
  DOC "SDL2 library"
)

# Let the user know there are problems finding installed headers.
# The user may not otherwise notice if they do not use the target.
if(NOT SDL2_LIBRARY)
  message(SEND_ERROR "Unable to find SDL2 library.")
endif()

# Let user use the headers via, say, `add_subdirectory`.
target_link_libraries(
  ${TARGET_NAME} INTERFACE $<BUILD_INTERFACE:${SDL2_LIBRARY}>
)

#
#
#
# ## Create a CMake package that imports all targets in this project.

# Generate the template needed to for creating the package file.
# This is done at configuration time.
set(TEMPLATE_DIR "${CMAKE_CURRENT_BINARY_DIR}/cmake_modules")
set(CONFIG_TEMPLATE "${TEMPLATE_DIR}/package_config_template.cmake.in")
set(CONFIG_INSTALL_DIR "${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake")
# Add targets to install into this.
# Every target in this list must include `${TARGET_NAME}-target.cmake`.
set(INSTALLED_BUILDSYSTEM_TARGETS "${PROJECT_NAME}")
# The above searches give absolute paths.
# Installed modules can only use relative paths.
# So the above fund include directories and libraries cannot be used.
# In the export modules, searches are repeated,
# but they should find the same files.
file(
  WRITE "${CONFIG_TEMPLATE}"
  [=[
@PACKAGE_INIT@

#
#
#
# ## Module dependencies

include(CMakeFindDependencyMacro)

foreach(TARGET_TO_INCLUDE @INSTALLED_BUILDSYSTEM_TARGETS@)
  include("${CMAKE_CURRENT_LIST_DIR}/${TARGET_TO_INCLUDE}-target.cmake")
endforeach()

#
# ### Headers

# Support `#include <SDL2/SDL*.h>`.
find_path(
  SDL2_INCLUDE_DIR "SDL2/SDL.h"
  PATH_SUFFIXES "include"
  DOC "SDL2 include directory"
)
if(SDL2_INCLUDE_DIR)
  target_include_directories(
    SDL2::SDL2 INTERFACE "${SDL2_INCLUDE_DIR}"
  )
endif()

# Support `#include <SDL*.h>`.
find_path(
  SDL2_INCLUDE_SUBDIR "SDL.h"
  PATH_SUFFIXES "include" "include/SDL2"
  DOC "SDL2 include subdirectory"
)
if(SDL2_INCLUDE_SUBDIR)
  target_include_directories(
    SDL2::SDL2 INTERFACE "${SDL2_INCLUDE_SUBDIR}"
  )
endif()

#
# ### Link dependencies

find_library(
  SDL2_LIBRARY
  NAMES SDL2
  PATH_SUFFIXES lib
  DOC "SDL2 library"
)
target_link_libraries(SDL2::SDL2 INTERFACE "${SDL2_LIBRARY}")

check_required_components(@INSTALLED_BUILDSYSTEM_TARGETS@)
]=]
)

# Tell CMake to generate the package.
set(CONFIG_FILENAME "${PROJECT_NAME}-config.cmake")
configure_package_config_file(
  # Tell CMake to use the template generated above.
  "${CONFIG_TEMPLATE}"
  # The file to contain the generated package file at build time.
  "${TEMPLATE_DIR}/${CONFIG_FILENAME}"
  # Directory to copy the package to at install time.
  # Note that it does not actually install the file created.
  INSTALL_DESTINATION
  "${CONFIG_INSTALL_DIR}"
)
# So install the file manually.
install(
  FILES "${TEMPLATE_DIR}/${CONFIG_FILENAME}"
  DESTINATION "${CONFIG_INSTALL_DIR}"
)
