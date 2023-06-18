cmake_minimum_required(VERSION 3.18 FATAL_ERROR)
cmake_policy(VERSION 3.18)

if (APPLE)
    set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64")
    set(CMAKE_OSX_DEPLOYMENT_TARGET "11.2")
endif()

set(NAME madrona-deps-compile)
project(${NAME})

include(FetchContent)
include(ExternalProject)

get_filename_component(PARENT_DIR ${CMAKE_SOURCE_DIR} DIRECTORY)
include(${PARENT_DIR}/cmake/sys-detect.cmake)

find_package(Python 3.6 COMPONENTS Interpreter)

include(versions.cmake)

if (MADRONA_LINUX)
    set (SDL_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/SDL")
    set (SDL_INSTALL "${SDL_PREFIX}/install")
    
    list(APPEND SDL_ARGS
        "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
        "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
        "-DCMAKE_INSTALL_PREFIX=${SDL_INSTALL}"
        "-DCMAKE_BUILD_TYPE=Release"
        "-DSDL_SHARED=ON"
        "-DSDL_STATIC=OFF"
        "-DSDL_ASSERTIONS=release"
        "-DSDL_WAYLAND=OFF"
        "-DSDL_KMSDRM=OFF"
        "-DSDL_OPENGLES=OFF"
        "-DSDL_DISKAUDIO=OFF"
        "-DSDL_X11=ON"
        "-DSDL_OPENGL=ON"
        "-DSDL_VULKAN=ON"
    )

    ExternalProject_Add(SDL
        GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
        GIT_TAG "${SDL_VERSION}"
        GIT_SHALLOW ON
        PREFIX "${SDL_PREFIX}"
        INSTALL_DIR "${SDL_INSTALL}"
        CMAKE_ARGS ${SDL_ARGS}
        DOWNLOAD_EXTRACT_TIMESTAMP TRUE
    )

    install(DIRECTORY "${SDL_INSTALL}/include/SDL2"
        DESTINATION SDL/include
    )
    
    install(DIRECTORY ${SDL_INSTALL}/lib/
        DESTINATION SDL/lib
        FILES_MATCHING
            PATTERN *
            PATTERN *pkgconfig* EXCLUDE
    )
endif()

set (VK_HDRS_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/vk_hdrs")
set (VK_HDRS_INSTALL "${VK_HDRS_PREFIX}/install")

list(APPEND VK_HDRS_ARGS
    "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
    "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
    "-DCMAKE_INSTALL_PREFIX=${VK_HDRS_INSTALL}"
    "-DCMAKE_BUILD_TYPE=Release"
)

if (MADRONA_MACOS OR MADRONA_IOS)
    list(APPEND VK_HDRS_ARGS
        -DCMAKE_OSX_ARCHITECTURES=arm64$<SEMICOLON>x86_64
        -DCMAKE_OSX_DEPLOYMENT_TARGET=11.2
    )
endif()

ExternalProject_Add(VulkanHeaders
    GIT_REPOSITORY https://github.com/KhronosGroup/Vulkan-Headers.git
    GIT_TAG sdk-${VULKAN_HEADER_VERSION}
    PREFIX "${VK_HDRS_PREFIX}"
    INSTALL_DIR "${VK_HDRS_INSTALL}"
    CMAKE_ARGS ${VK_HDRS_ARGS}
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE
)

install(DIRECTORY ${VK_HDRS_INSTALL}/include/vulkan
    DESTINATION vk_hdrs/include/
)

set(VK_LOADER_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/vk_loader")
set(VK_LOADER_INSTALL "${VK_LOADER_PREFIX}/install")

list(APPEND VK_LOADER_ARGS
    "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
    "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
    "-DCMAKE_INSTALL_PREFIX=${VK_LOADER_INSTALL}"
    "-DCMAKE_BUILD_TYPE=Release"
    "-DUPDATE_DEPS=ON"
)

if (MADRONA_MACOS OR MADRONA_IOS)
    list(APPEND VK_LOADER_ARGS
        -DCMAKE_OSX_ARCHITECTURES=arm64$<SEMICOLON>x86_64
        -DCMAKE_OSX_DEPLOYMENT_TARGET=11.2
    )
endif()

ExternalProject_Add(VulkanLoader
    GIT_REPOSITORY https://github.com/KhronosGroup/Vulkan-Loader.git
    GIT_TAG sdk-${VULKAN_LOADER_VERSION}
    PREFIX "${VK_LOADER_PREFIX}"
    INSTALL_DIR "${VK_LOADER_INSTALL}"
    CMAKE_ARGS ${VK_LOADER_ARGS}
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE
)

file(GLOB vk_loader_libs ${VK_LOADER_INSTALL}/lib/libvulkan*)

install(FILES ${vk_loader_libs}
    DESTINATION vk_loader/lib
    USE_SOURCE_PERMISSIONS
)

if (MADRONA_MACOS)
    set (MOLTENVK_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/moltenvk")

    ExternalProject_Add(MoltenVK
        GIT_REPOSITORY https://github.com/KhronosGroup/MoltenVK.git
        GIT_TAG ${MOLTENVK_VERSION}
        PREFIX ${MOLTENVK_PREFIX}
        STAMP_DIR ${MOLTENVK_PREFIX}/stamps
        SOURCE_DIR ${MOLTENVK_PREFIX}/repo
        BINARY_DIR ${MOLTENVK_PREFIX}/repo
        CONFIGURE_COMMAND ""
        BUILD_COMMAND
            ./fetchDependencies --macos
            COMMAND make MVK_CONFIG_LOG_LEVEL=1 MVK_HIDE_VULKAN_SYMBOLS=1 macos
        INSTALL_COMMAND ""
    )

    file(GLOB moltenvk_libs ${MOLTENVK_PREFIX}/repo/Package/Release/MoltenVK/dylib/macOS/*)

    install(FILES ${moltenvk_libs}
        DESTINATION moltenvk/lib/
        USE_SOURCE_PERMISSIONS
    )

    install(DIRECTORY ${MOLTENVK_PREFIX}/repo/Package/Release/MoltenVK/include/MoltenVK
        DESTINATION moltenvk/include/
    )

    FetchContent_Declare(metal-cpp
        URL "https://developer.apple.com/metal/cpp/files/metal-cpp_macOS13_iOS16.zip"
        URL_HASH SHA256=6f741894229e9c750add1afc3797274fc008c7507e2ae726370c17c34b7c6a68
        DOWNLOAD_EXTRACT_TIMESTAMP TRUE
    )
    FetchContent_MakeAvailable(metal-cpp)

    install(DIRECTORY ${metal-cpp_SOURCE_DIR}/Foundation
        DESTINATION metal-cpp/include
    )
    install(DIRECTORY ${metal-cpp_SOURCE_DIR}/Metal
        DESTINATION metal-cpp/include
    )
    install(DIRECTORY ${metal-cpp_SOURCE_DIR}/QuartzCore
        DESTINATION metal-cpp/include
    )
    install(FILES ${metal-cpp_SOURCE_DIR}/LICENSE.txt
        DESTINATION metal-cpp
    )

    # Want metal-cpp-extensions from this example
    # (why is it not distributed separately??)
    FetchContent_Declare(metal-cpp-extensions
        URL "https://developer.apple.com/metal/LearnMetalCPP.zip"
        URL_HASH SHA256=a709f3c0b532d5b9e3a5db3da3a35f3b783af27eb50f23a711115c02f86a256d
        DOWNLOAD_EXTRACT_TIMESTAMP TRUE
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
    )
    FetchContent_MakeAvailable(metal-cpp-extensions)

    set(metal_cpp_ext_dir "${metal-cpp-extensions_SOURCE_DIR}/metal-cpp-extensions")
    install(DIRECTORY "${metal_cpp_ext_dir}/AppKit"
        DESTINATION metal-cpp-extensions/include
    )
    install(DIRECTORY "${metal_cpp_ext_dir}/MetalKit"
        DESTINATION metal-cpp-extensions/include
    )

endif ()
