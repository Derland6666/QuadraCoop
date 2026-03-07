# ================================================================
#  QuadroCoop — CMakeLists.txt
#  Geode SDK 4.x mod build configuration
#
#  Requirements:
#    - CMake 3.21 or higher
#    - GEODE_SDK environment variable pointing to your Geode SDK clone
#      (cloned from https://github.com/geode-sdk/geode)
#    - C++20 compiler: MSVC 2019+, Clang 13+, or GCC 11+
#
#  Expected project layout:
#    QuadroCoop/
#    ├── CMakeLists.txt      <- this file
#    ├── mod.json            <- mod manifest (MUST be in this folder)
#    ├── about.md            <- mod description shown in the Geode index
#    ├── changelog.md        <- optional; auto-included if present
#    ├── logo.png            <- 128x128 PNG; shown in the mod list
#    ├── resources/          <- optional; sprites, audio, fonts, etc.
#    │   └── ...
#    └── src/
#        └── main.cpp
#
#  Build:
#    cmake -B build
#    cmake --build build --config RelWithDebInfo
#
#  Output:
#    build/QuadroCoop.geode  — ready to drop into your mods folder
#                              or submit to the Geode mod index.
# ================================================================

cmake_minimum_required(VERSION 3.21)

# ── C++ standard — must match Geode SDK requirement (C++20) ──────
set(CMAKE_CXX_STANDARD          20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Hide symbols by default so they don't leak between mods loaded
# in the same process. Required for Android; good practice everywhere.
set(CMAKE_CXX_VISIBILITY_PRESET  hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

project(QuadroCoop VERSION 2.0.0)

# ── Locate Geode SDK ─────────────────────────────────────────────
# The GEODE_SDK environment variable must point to your local clone
# of the Geode SDK repository.  The CLI command `geode sdk install`
# sets this variable automatically.
if (NOT DEFINED ENV{GEODE_SDK})
    message(FATAL_ERROR
        "[QuadroCoop] GEODE_SDK is not set.\n"
        "Run 'geode sdk install' or set the variable manually:\n"
        "  Windows : set GEODE_SDK=C:\\path\\to\\geode\n"
        "  macOS   : export GEODE_SDK=/path/to/geode\n"
        "  Linux   : export GEODE_SDK=/path/to/geode\n"
    )
endif()

message(STATUS "[QuadroCoop] Using Geode SDK at: $ENV{GEODE_SDK}")

# Add the SDK as a subdirectory. The binary dir keeps SDK build
# artifacts isolated from the mod's own build artifacts.
add_subdirectory($ENV{GEODE_SDK} ${CMAKE_CURRENT_BINARY_DIR}/geode-sdk-build)


# ================================================================
#  Mod target
# ================================================================

add_library(${PROJECT_NAME} SHARED
    src/main.cpp

    # Add additional translation units here as the project grows:
    # src/TriggerSystem.cpp
    # src/SettingsPopup.cpp
    # src/PortalLogic.cpp
)

# Private include path so #include "Something.hpp" works from src/.
target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# ── Compiler warnings ────────────────────────────────────────────
# /W3 on MSVC; -Wall -Wextra on Clang/GCC.
# -Wno-unused-parameter suppresses noise from Geode hook overrides
# that intentionally ignore some parameters.
if (MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE
        /W3
        /wd4068  # suppress "unknown pragma" warnings from Geode macros
    )
else()
    target_compile_options(${PROJECT_NAME} PRIVATE
        -Wall
        -Wextra
        -Wno-unused-parameter
    )
endif()

# ── Link Geode SDK ───────────────────────────────────────────────
# geode-sdk is the single link target that brings in all Cocos2d,
# GD headers, and Geode runtime dependencies transitively.
# Do NOT also link cocos2d or libExtensions separately — this causes
# duplicate symbol errors on Android and macOS.
target_link_libraries(${PROJECT_NAME} geode-sdk)


# ================================================================
#  Packaging — create_geode_file
#
#  This Geode SDK CMake function:
#    1. Reads mod.json from CMAKE_CURRENT_SOURCE_DIR (project root).
#    2. Validates the manifest schema for the target Geode version.
#    3. Bundles the compiled binary, mod.json, about.md (if present),
#       changelog.md (if present), logo.png (if present), and the
#       entire resources/ directory (if present) into a .geode ZIP.
#    4. Writes the archive to the build output directory as
#       <mod-id>.geode  (e.g. yourname.quadrocoop.geode).
#
#  Both mod.json and about.md MUST be in CMAKE_CURRENT_SOURCE_DIR
#  (i.e. the same directory as this CMakeLists.txt) for the function
#  to locate them automatically.  No explicit file list is required.
#
#  To install directly to your local GD mods folder after building,
#  set the GEODE_INSTALL_MOD environment variable or use:
#    cmake --build build --target install-QuadroCoop
# ================================================================

# Tell CMake (and Geode's packaging step) exactly where mod.json is.
# This property is read by create_geode_file() in Geode SDK 3.x/4.x.
set_target_properties(${PROJECT_NAME} PROPERTIES
    GEODE_MOD_JSON "${CMAKE_CURRENT_SOURCE_DIR}/mod.json"
)

# Package everything into a .geode archive.
# Must be called AFTER target_link_libraries.
create_geode_file(${PROJECT_NAME})
