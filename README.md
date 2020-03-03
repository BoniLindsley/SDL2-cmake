# SDL2 CMake config package

This package is intended for distributions
that do not provide an `SDL2` CMake package.


## Usage

This package does not build nor install SDL2,
but instead installs an `SDL2::SDL2` target for use in CMake scripts.
So it expects the SDL2 library to already be installed.

Rough description:

  * CMake configure and generate to ensure SDL2 can be found.
  * CMake build and install.
  * Use `find_package(SDL2)`
    and `target_link_libraries(${TARGET_NAME} SDL2::SDL2)`
    in CMake scripts.
  * It may also be possible to use `add_subdirectory` on this directory,
    and `target_link_libraries(${TARGET_NAME} SDL2)`.

Note that there is no attempt to provide components
-- I only added features that I needed.


## Purpose

CMake does not provide a SDL2 package by default.
Distributions also may not provide one.

It may also used for bridging header `#include` directives
between `<SDL*.h>` and `<SDL2/SDL*.h>`.
This may become necessary because there is no universal install path.
For example, Emscripten usees the former whereas Debian uses the latter.
Source code may also use either.
