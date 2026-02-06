cmake_minimum_required(VERSION 3.16)
project(RGFW VERSION 2.0.0 LANGUAGES C)

# Default install prefix to local "install" folder if not specified
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "Install path prefix" FORCE)
endif()

# =============================================================================
# Options
# =============================================================================
option(RGFW_BUILD_EXAMPLES "Build RGFW examples" ON)
option(RGFW_BUILD_SHARED "Build RGFW as a shared library (OFF for static)" ON)

# Platform-specific options
option(RGFW_USE_WAYLAND "Use Wayland instead of X11 (Linux only)" OFF)
option(RGFW_USE_WAYLAND_X11 "Use Wayland with X11 fallback (Linux only)" OFF)

# Feature toggles for examples
option(RGFW_NO_VULKAN "Disable Vulkan example" OFF)
option(RGFW_NO_GLES "Disable GLES example" OFF)
option(RGFW_NO_OSMESA "Disable OSMesa example" OFF)
option(RGFW_NO_EGL "Disable EGL example" OFF)

# =============================================================================
# Platform detection
# =============================================================================
if(WIN32)
    set(RGFW_PLATFORM "WINDOWS")
elseif(APPLE)
    set(RGFW_PLATFORM "MACOS")
elseif(UNIX)
    set(RGFW_PLATFORM "LINUX")
elseif(EMSCRIPTEN)
    set(RGFW_PLATFORM "WASM")
else()
    message(FATAL_ERROR "Unsupported platform")
endif()

message(STATUS "RGFW: Detected platform: ${RGFW_PLATFORM}")

# Auto-disable certain features on non-Linux platforms
if(NOT RGFW_PLATFORM STREQUAL "LINUX")
    set(RGFW_NO_GLES ON)
    set(RGFW_NO_OSMESA ON)
    set(RGFW_NO_EGL ON)
    set(RGFW_USE_WAYLAND OFF)
    set(RGFW_USE_WAYLAND_X11 OFF)
endif()

# Wayland implies no Vulkan example (for now)
if(RGFW_USE_WAYLAND OR RGFW_USE_WAYLAND_X11)
    set(RGFW_NO_VULKAN ON)
endif()

# =============================================================================
# Find packages
# =============================================================================
if(RGFW_PLATFORM STREQUAL "WINDOWS")
    # Windows doesn't need find_package for system libs
elseif(RGFW_PLATFORM STREQUAL "MACOS")
    find_library(COCOA_FRAMEWORK Cocoa REQUIRED)
    find_library(OPENGL_FRAMEWORK OpenGL REQUIRED)
    find_library(IOKIT_FRAMEWORK IOKit REQUIRED)
    find_library(COREVIDEO_FRAMEWORK CoreVideo REQUIRED)
    find_library(METAL_FRAMEWORK Metal)
    find_library(QUARTZCORE_FRAMEWORK QuartzCore)
elseif(RGFW_PLATFORM STREQUAL "LINUX")
    find_package(X11)
    find_package(OpenGL)

    if(RGFW_USE_WAYLAND OR RGFW_USE_WAYLAND_X11)
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(WAYLAND REQUIRED wayland-client wayland-cursor wayland-egl xkbcommon)
        pkg_check_modules(EGL REQUIRED egl)
    endif()
endif()

# =============================================================================
# RGFW Library (Interface / Header-only by default, or compiled)
# =============================================================================

# Create RGFW.c from RGFW.h for compiled library
set(RGFW_GENERATED_SOURCE "${CMAKE_CURRENT_BINARY_DIR}/RGFW.c")
file(WRITE ${RGFW_GENERATED_SOURCE} "#define RGFW_IMPLEMENTATION\n#include \"RGFW.h\"\n")

# Common compile definitions
set(RGFW_COMPILE_DEFINITIONS RGFW_IMPLEMENTATION RGFW_OPENGL)

# Platform-specific definitions
if(RGFW_PLATFORM STREQUAL "LINUX")
    if(RGFW_USE_WAYLAND AND RGFW_USE_WAYLAND_X11)
        list(APPEND RGFW_COMPILE_DEFINITIONS RGFW_WAYLAND RGFW_X11)
    elseif(RGFW_USE_WAYLAND)
        list(APPEND RGFW_COMPILE_DEFINITIONS RGFW_WAYLAND)
    endif()
endif()

# Common include directories
set(RGFW_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")

# Include GNUInstallDirs early for install paths
include(GNUInstallDirs)

# Function to setup RGFW library target
function(setup_rgfw_target target_name)
    target_include_directories(${target_name} PUBLIC
        $<BUILD_INTERFACE:${RGFW_INCLUDE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    )
    target_compile_definitions(${target_name}
        PRIVATE ${RGFW_COMPILE_DEFINITIONS} RGFW_EXPORT
        PUBLIC RGFW_NATIVE
    )

    target_sources(${target_name} PRIVATE RGFW.c)

    if(RGFW_PLATFORM STREQUAL "WINDOWS")
        target_link_libraries(${target_name} PUBLIC gdi32 shell32 user32 advapi32)
        target_compile_definitions(${target_name} PRIVATE _WIN32_WINNT=0x0501)
    elseif(RGFW_PLATFORM STREQUAL "MACOS")
        target_link_libraries(${target_name} PUBLIC
            ${COCOA_FRAMEWORK}
            ${OPENGL_FRAMEWORK}
            ${IOKIT_FRAMEWORK}
            ${COREVIDEO_FRAMEWORK}
        )
    elseif(RGFW_PLATFORM STREQUAL "LINUX")
        if(RGFW_USE_WAYLAND OR RGFW_USE_WAYLAND_X11)
            target_link_libraries(${target_name} PUBLIC
                ${WAYLAND_LIBRARIES}
                ${EGL_LIBRARIES}
                GL m
            )
            target_include_directories(${target_name} PRIVATE ${WAYLAND_INCLUDE_DIRS} ${EGL_INCLUDE_DIRS})
            if(RGFW_USE_WAYLAND_X11)
                target_link_libraries(${target_name} PUBLIC X11 Xrandr)
            endif()
        else()
            target_link_libraries(${target_name} PUBLIC X11 Xrandr GL dl pthread m)
        endif()
    endif()

    # Release optimizations: enable dead-code stripping
    if(MSVC)
        target_compile_options(${target_name} PRIVATE
            $<$<NOT:$<CONFIG:Debug>>:/Gy>
        )
        target_link_options(${target_name} PRIVATE
            $<$<NOT:$<CONFIG:Debug>>:/OPT:REF /OPT:ICF>
        )
    else()
        target_compile_options(${target_name} PRIVATE
            $<$<NOT:$<CONFIG:Debug>>:-ffunction-sections -fdata-sections>
        )
        if(APPLE)
            target_link_options(${target_name} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:-Wl,-dead_strip>
            )
        else()
            target_link_options(${target_name} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:-Wl,--gc-sections>
            )
        endif()
    endif()

    # Strip debug symbols from release shared libraries
    if(NOT MSVC)
        get_target_property(_target_type ${target_name} TYPE)
        if(_target_type STREQUAL "SHARED_LIBRARY")
            if(APPLE)
                target_link_options(${target_name} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-Wl,-x>
                )
            else()
                target_link_options(${target_name} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-Wl,-s>
                )
            endif()
        endif()
    endif()

    set_target_properties(${target_name} PROPERTIES
        C_STANDARD 99
        C_STANDARD_REQUIRED ON
        POSITION_INDEPENDENT_CODE ON
    )
endfunction()

# RGFW library — always named "RGFW", type controlled by RGFW_BUILD_SHARED
if(RGFW_BUILD_SHARED)
    add_library(RGFW SHARED ${RGFW_GENERATED_SOURCE})
    set_target_properties(RGFW PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
    )
else()
    add_library(RGFW STATIC ${RGFW_GENERATED_SOURCE})
endif()
setup_rgfw_target(RGFW)
add_library(RGFW::RGFW ALIAS RGFW)

# Header-only interface library (for single-header usage)
add_library(RGFW_header INTERFACE)
target_include_directories(RGFW_header INTERFACE ${RGFW_INCLUDE_DIR})
add_library(RGFW::header ALIAS RGFW_header)

# =============================================================================
# Examples
# =============================================================================
if(RGFW_BUILD_EXAMPLES)
    message(STATUS "RGFW: Building examples")

    # Helper function to create an example
    function(add_rgfw_example name source_file)
        cmake_parse_arguments(EXAMPLE
            ""
            "OUTPUT_NAME"
            "EXTRA_SOURCES;DEFINITIONS;LIBRARIES;FRAMEWORKS"
            ${ARGN}
        )

        add_executable(${name} ${source_file} ${EXAMPLE_EXTRA_SOURCES})

        # Link against the RGFW library (brings includes + platform libs via PUBLIC)
        target_link_libraries(${name} PRIVATE RGFW)
        target_compile_definitions(${name} PRIVATE ${EXAMPLE_DEFINITIONS})

        if(EXAMPLE_OUTPUT_NAME)
            set_target_properties(${name} PROPERTIES OUTPUT_NAME ${EXAMPLE_OUTPUT_NAME})
        endif()

        # Filter out 'm' library on Windows (math is part of C runtime on MSVC)
        set(FILTERED_LIBRARIES ${EXAMPLE_LIBRARIES})
        if(RGFW_PLATFORM STREQUAL "WINDOWS")
            list(REMOVE_ITEM FILTERED_LIBRARIES m)
        endif()

        # Extra libraries beyond what RGFW provides
        if(FILTERED_LIBRARIES)
            target_link_libraries(${name} PRIVATE ${FILTERED_LIBRARIES})
        endif()

        if(EXAMPLE_FRAMEWORKS)
            foreach(fw ${EXAMPLE_FRAMEWORKS})
                find_library(${fw}_LIB ${fw})
                if(${fw}_LIB)
                    target_link_libraries(${name} PRIVATE ${${fw}_LIB})
                endif()
            endforeach()
        endif()

        set_target_properties(${name} PROPERTIES
            C_STANDARD 99
            C_STANDARD_REQUIRED ON
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
        )
    endfunction()

    # =========================================================================
    # Basic examples (all platforms)
    # =========================================================================
    set(BASIC_EXAMPLES
        gl11
        surface
        event_queue
        callbacks
        state-checking
        flags
        monitor
        gamma
        gl33_ctx
        smooth-resize
        multi-window
        custom_alloc
        flash
    )

    foreach(example ${BASIC_EXAMPLES})
        if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/${example}/${example}.c")
            add_rgfw_example(example_${example}
                "examples/${example}/${example}.c"
                OUTPUT_NAME ${example}
                DEFINITIONS RGFW_OPENGL
            )
        endif()
    endforeach()

    # Standard mouse icons (different output name)
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/standard-mouse-icons/icons.c")
        add_rgfw_example(example_standard_mouse_icons
            "examples/standard-mouse-icons/icons.c"
            OUTPUT_NAME standard-mouse-icons
            DEFINITIONS RGFW_OPENGL
        )
    endif()

    # =========================================================================
    # Custom examples
    # =========================================================================

    # Window icons
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/window_icons/icons.c")
        add_rgfw_example(example_window_icons
            "examples/window_icons/icons.c"
            OUTPUT_NAME window_icons
            DEFINITIONS RGFW_OPENGL
            LIBRARIES m
        )
    endif()

    # Mouse icons
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/mouse_icons/icons.c")
        add_rgfw_example(example_mouse_icons
            "examples/mouse_icons/icons.c"
            OUTPUT_NAME mouse_icons
            DEFINITIONS RGFW_OPENGL
            LIBRARIES m
        )
    endif()

    # Gamepad - disabled due to build issues
    # if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/gamepad/gamepad.c")
    #     add_rgfw_example(example_gamepad
    #         "examples/gamepad/gamepad.c"
    #         OUTPUT_NAME gamepad
    #         DEFINITIONS RGFW_OPENGL
    #         LIBRARIES m
    #     )
    # endif()

    # First-person camera
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/first-person-camera/camera.c")
        add_rgfw_example(example_camera
            "examples/first-person-camera/camera.c"
            OUTPUT_NAME camera
            DEFINITIONS RGFW_OPENGL
            LIBRARIES m
        )
    endif()

    # GL33
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/gl33/gl33.c")
        add_executable(example_gl33 "examples/gl33/gl33.c")
        target_link_libraries(example_gl33 PRIVATE RGFW)
        target_compile_definitions(example_gl33 PRIVATE RGFW_OPENGL)
        set_target_properties(example_gl33 PROPERTIES
            OUTPUT_NAME gl33
            C_STANDARD 99
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
        )
        if(RGFW_PLATFORM STREQUAL "WINDOWS")
            target_compile_definitions(example_gl33 PRIVATE UNICODE)
        elseif(RGFW_PLATFORM STREQUAL "LINUX")
            if(RGFW_USE_WAYLAND OR RGFW_USE_WAYLAND_X11)
                target_link_libraries(example_gl33 PRIVATE wayland-egl m)
            else()
                target_link_libraries(example_gl33 PRIVATE m)
            endif()
        endif()
    endif()

    # Gears
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/gears/gears.c")
        add_rgfw_example(example_gears
            "examples/gears/gears.c"
            OUTPUT_NAME gears
            DEFINITIONS RGFW_OPENGL
            LIBRARIES m
        )
    endif()

    # sRGB
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/srgb/srgb.c")
        add_rgfw_example(example_srgb
            "examples/srgb/srgb.c"
            OUTPUT_NAME srgb
            DEFINITIONS RGFW_OPENGL
            LIBRARIES m
        )
    endif()

    # PortableGL (not for emscripten)
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/portableGL/pgl.c" AND NOT EMSCRIPTEN)
        add_executable(example_portableGL "examples/portableGL/pgl.c")
        target_link_libraries(example_portableGL PRIVATE RGFW)
        # Suppress warnings (different flag for MSVC vs GCC/Clang)
        if(MSVC)
            target_compile_options(example_portableGL PRIVATE /W0)
        else()
            target_compile_options(example_portableGL PRIVATE -w)
        endif()
        set_target_properties(example_portableGL PROPERTIES
            OUTPUT_NAME pgl
            C_STANDARD 99
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
        )
    endif()

    # MicroUI demo
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/microui_demo/microui_demo.c")
        add_rgfw_example(example_microui
            "examples/microui_demo/microui_demo.c"
            OUTPUT_NAME microui_demo
            DEFINITIONS RGFW_OPENGL
            EXTRA_SOURCES "examples/microui_demo/microui.c"
        )
    endif()

    # Minimal links
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/minimal_links/minimal_links.c")
        add_executable(example_minimal_links "examples/minimal_links/minimal_links.c")
        target_link_libraries(example_minimal_links PRIVATE RGFW)
        set_target_properties(example_minimal_links PROPERTIES
            OUTPUT_NAME minimal_links
            C_STANDARD 99
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
        )
    endif()

    # =========================================================================
    # Platform-specific examples
    # =========================================================================

    # DirectX 11 (Windows only)
    if(RGFW_PLATFORM STREQUAL "WINDOWS" AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/dx11/dx11.c")
        add_executable(example_dx11 "examples/dx11/dx11.c")
        target_link_libraries(example_dx11 PRIVATE RGFW d3d11 dxgi d3dcompiler uuid)
        target_compile_definitions(example_dx11 PRIVATE RGFW_DIRECTX)
        set_target_properties(example_dx11 PROPERTIES
            OUTPUT_NAME dx11
            C_STANDARD 99
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
        )
    endif()

    # Metal (macOS only)
    if(RGFW_PLATFORM STREQUAL "MACOS" AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/metal/metal.m")
        add_executable(example_metal "examples/metal/metal.m")
        target_link_libraries(example_metal PRIVATE RGFW
            ${METAL_FRAMEWORK}
            ${QUARTZCORE_FRAMEWORK}
        )
        set_target_properties(example_metal PROPERTIES
            OUTPUT_NAME metal
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
        )
    endif()

    # GLES2 (Linux only, non-Wayland-only)
    if(NOT RGFW_NO_GLES AND RGFW_PLATFORM STREQUAL "LINUX")
        if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/gles2/gles2.c")
            add_executable(example_gles2 "examples/gles2/gles2.c")
            target_link_libraries(example_gles2 PRIVATE RGFW GL EGL)
            target_compile_definitions(example_gles2 PRIVATE RGFW_EGL)
            set_target_properties(example_gles2 PROPERTIES
                OUTPUT_NAME gles2
                C_STANDARD 99
                RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
            )
        endif()
    endif()

    # EGL (Linux only)
    if(NOT RGFW_NO_EGL AND RGFW_PLATFORM STREQUAL "LINUX")
        if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/egl/egl.c")
            add_executable(example_egl "examples/egl/egl.c")
            target_link_libraries(example_egl PRIVATE RGFW GL EGL)
            target_compile_definitions(example_egl PRIVATE RGFW_EGL RGFW_OPENGL)
            set_target_properties(example_egl PROPERTIES
                OUTPUT_NAME egl
                C_STANDARD 99
                RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
            )
        endif()
    endif()

    # OSMesa (Linux only)
    if(NOT RGFW_NO_OSMESA AND RGFW_PLATFORM STREQUAL "LINUX")
        if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/osmesa_demo/osmesa_demo.c")
            find_library(OSMESA_LIBRARY OSMesa)
            if(OSMESA_LIBRARY)
                add_executable(example_osmesa "examples/osmesa_demo/osmesa_demo.c")
                target_link_libraries(example_osmesa PRIVATE RGFW ${OSMESA_LIBRARY})
                set_target_properties(example_osmesa PROPERTIES
                    OUTPUT_NAME osmesa_demo
                    C_STANDARD 99
                    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
                )
            else()
                message(STATUS "RGFW: OSMesa not found, skipping osmesa_demo example")
            endif()
        endif()
    endif()

    # Vulkan (requires Vulkan SDK)
    if(NOT RGFW_NO_VULKAN AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/vk10/vk10.c")
        find_package(Vulkan)
        if(Vulkan_FOUND)
            # Check for shader files
            set(VERT_SHADER "${CMAKE_CURRENT_SOURCE_DIR}/examples/vk10/shaders/vert.h")
            set(FRAG_SHADER "${CMAKE_CURRENT_SOURCE_DIR}/examples/vk10/shaders/frag.h")

            if(EXISTS "${VERT_SHADER}" AND EXISTS "${FRAG_SHADER}")
                add_executable(example_vulkan "examples/vk10/vk10.c")
                target_link_libraries(example_vulkan PRIVATE RGFW Vulkan::Vulkan)
                target_include_directories(example_vulkan PRIVATE ${Vulkan_INCLUDE_DIRS})
                target_compile_definitions(example_vulkan PRIVATE RGFW_VULKAN)

                set_target_properties(example_vulkan PROPERTIES
                    OUTPUT_NAME vk10
                    C_STANDARD 99
                    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
                )
            else()
                message(STATUS "RGFW: Vulkan shaders not found. Run glslangValidator to compile shaders first.")
                message(STATUS "  glslangValidator -V examples/vk10/shaders/vert.vert -o examples/vk10/shaders/vert.h --vn vert_code")
                message(STATUS "  glslangValidator -V examples/vk10/shaders/frag.frag -o examples/vk10/shaders/frag.h --vn frag_code")
            endif()
        else()
            message(STATUS "RGFW: Vulkan not found, skipping vk10 example")
        endif()
    endif()

endif() # RGFW_BUILD_EXAMPLES

# =============================================================================
# Installation
# =============================================================================

# Install header
install(FILES RGFW.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# Install library
install(TARGETS RGFW
    EXPORT RGFWTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# Export targets
install(EXPORT RGFWTargets
    FILE RGFWTargets.cmake
    NAMESPACE RGFW::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/RGFW
)

# Create config file
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/RGFWConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/RGFWConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/RGFWConfig.cmake"
    @ONLY
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/RGFWConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/RGFWConfigVersion.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/RGFW
)

# =============================================================================
# Summary
# =============================================================================
message(STATUS "")
message(STATUS "RGFW Configuration Summary:")
message(STATUS "  Platform:        ${RGFW_PLATFORM}")
if(RGFW_BUILD_SHARED)
    message(STATUS "  Library type:    SHARED")
else()
    message(STATUS "  Library type:    STATIC")
endif()
message(STATUS "  Build examples:  ${RGFW_BUILD_EXAMPLES}")
if(RGFW_PLATFORM STREQUAL "LINUX")
    message(STATUS "  Use Wayland:     ${RGFW_USE_WAYLAND}")
    message(STATUS "  Wayland + X11:   ${RGFW_USE_WAYLAND_X11}")
endif()
message(STATUS "")
