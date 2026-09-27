cmake_minimum_required(VERSION 3.15)
project(ChordAnalyzer VERSION 1.0.0)

# Enable CTest so per-plugin regression tests registered via add_test() are
# discoverable from the build root (`ctest --test-dir build`).
enable_testing()

# C++ standard is set per-plugin (most use C++17, set in their own CMakeLists.txt)

# Include global settings
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/GlobalSettings.cmake)

# Include common JUCE defaults
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/JuceDefaults.cmake)

# Include CLAP helper (apply_clap_extension)
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/ClapDefaults.cmake)

# Build system notes:
# - JUCE path: configurable via JUCE_PATH variable
# - Default: ../JUCE (sibling directory for local dev)
# - CI builds pass -DJUCE_PATH=/path/to/JUCE
# - Plugins install to ~/.vst3/ and ~/.lv2/

# JUCE path configuration
# Priority: 1) JUCE_PATH cmake variable, 2) sibling directory, 3) external/JUCE submodule
if(DEFINED JUCE_PATH)
    # Use explicitly provided path (CI builds)
    message(STATUS "Using JUCE from: ${JUCE_PATH}")
    add_subdirectory(${JUCE_PATH} ${CMAKE_CURRENT_BINARY_DIR}/JUCE)
elseif(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/../JUCE/CMakeLists.txt")
    # Local development: sibling directory
    message(STATUS "Using JUCE from sibling directory: ${CMAKE_CURRENT_SOURCE_DIR}/../JUCE")
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../JUCE ${CMAKE_CURRENT_BINARY_DIR}/JUCE)
elseif(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/external/JUCE/CMakeLists.txt")
    # Git submodule
    message(STATUS "Using JUCE from submodule: external/JUCE")
    add_subdirectory(external/JUCE)
else()
    message(FATAL_ERROR "JUCE not found! Options:\n"
        "  1. Set -DJUCE_PATH=/path/to/JUCE\n"
        "  2. Clone JUCE to ../JUCE (sibling directory)\n"
        "  3. Add as submodule: git submodule add https://github.com/juce-framework/JUCE.git external/JUCE")
endif()

# CLAP format support (clap-juce-extensions wrapper around JUCE plugins).
# When ON we pull in the submodule under external/clap-juce-extensions; if the
# submodule isn't checked out we silently fall back to OFF so local devs who
# haven't run `git submodule update` can still build VST3/AU/LV2.
option(BUILD_CLAP "Build CLAP format for all plugins" ON)
if(BUILD_CLAP)
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/external/clap-juce-extensions/CMakeLists.txt")
        message(STATUS "CLAP: using external/clap-juce-extensions")
        add_subdirectory(external/clap-juce-extensions EXCLUDE_FROM_ALL)
    else()
        message(WARNING "BUILD_CLAP=ON but external/clap-juce-extensions not found — disabling CLAP. "
            "Run: git submodule update --init --recursive")
        set(BUILD_CLAP OFF CACHE BOOL "Build CLAP format for all plugins" FORCE)
    endif()
endif()

# CI builds don't need to install plugins to system directories
option(DUSK_COPY_AFTER_BUILD "Copy plugins to system VST3/LV2 dirs after build" ON)

# Build Chord Analyzer plugin
option(BUILD_CHORD_ANALYZER "Build Chord Analyzer plugin" ON)

# Add plugin subdirectory
if(BUILD_CHORD_ANALYZER)
    add_subdirectory(plugins/chord-analyzer)
endif()
