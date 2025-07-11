cmake_minimum_required(VERSION 3.22)
project(CImgPlugin VERSION 0.1)

set(HEADER_FILES
    src/CImgPlugin/ImageCImg.h
    src/CImgPlugin/CImgPlugin.h.in
    src/CImgPlugin/SOFACImg.h
    )

set(SOURCE_FILES
    src/CImgPlugin/ImageCImg.cpp
    src/CImgPlugin/initCImgPlugin.cpp
    )

sofa_find_package(Sofa.Helper REQUIRED)
sofa_find_package(Sofa.DefaultType REQUIRED)

# add FindCImg.cmake
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindCImg.cmake ${CMAKE_BINARY_DIR}/lib/cmake/FindCImg.cmake COPYONLY)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindCImg.cmake DESTINATION lib/cmake/${PROJECT_NAME} COMPONENT headers)

sofa_find_package(CImg QUIET)

if(NOT CImg_FOUND AND SOFA_ALLOW_FETCH_DEPENDENCIES)
    message("CImgPlugin: DEPENDENCY CImg NOT FOUND. SOFA_ALLOW_FETCH_DEPENDENCIES is ON, fetching CImg...")

    include(FetchContent)
    FetchContent_Declare(CImg
            GIT_REPOSITORY https://github.com/GreycLab/CImg
            GIT_TAG        v.3.3.2
    )

    FetchContent_GetProperties(CImg)
    if(NOT CImg_POPULATED)
        FetchContent_Populate(CImg)
        set(CIMG_DIR ${cimg_SOURCE_DIR})
        sofa_find_package(CImg REQUIRED)
    endif()
elseif (NOT CImg_FOUND)
    message(FATAL_ERROR "CImgPlugin: DEPENDENCY CImg NOT FOUND. SOFA_ALLOW_FETCH_DEPENDENCIES is OFF and thus cannot be fetched. Install cimg-dev, or enable SOFA_ALLOW_FETCH_DEPENDENCIES to fix this issue.")
endif()

# OS X only: if the user installed its own JPEG/PNG lib (typically with homebrew/port),
# it will allow to use those instead of those present in XCode's frameworks
set(CMAKE_FIND_FRAMEWORK LAST)

sofa_find_package(TIFF QUIET)
sofa_find_package(ZSTD QUIET)
sofa_find_package(JPEG QUIET)
sofa_find_package(PNG QUIET)
sofa_find_package(ZLIB QUIET)

# Configure Paths according to detected packages
set(DEP_TARGETS "")
set(CIMG_CFLAGS -Dcimg_no_system_calls) # Remove CImg sys calls
if(TIFF_FOUND AND ( NOT(SOFA_BUILD_RELEASE_PACKAGE) OR ZSTD_FOUND))
    message(STATUS "CImgPlugin: TIFF Image Format enabled")
    list(APPEND CIMG_CFLAGS -Dcimg_use_tiff)
    list(APPEND DEP_TARGETS TIFF::TIFF)
    if(ZSTD_FOUND)
        set(DEP_ZSTD ZSTD::ZSTD)
    endif()
else()
    message(STATUS "CImgPlugin: TIFF Image Format disabled")
endif()
if(JPEG_FOUND)
    message(STATUS "CImgPlugin: JPEG Image Format enabled")
    list(APPEND CIMG_CFLAGS -Dcimg_use_jpeg)
    if(WIN32)
        # prevent INT32 redefinition on Windows (see jmorecfg.h)
        list(APPEND CIMG_CFLAGS -DXMD_H)
    endif()
    list(APPEND DEP_TARGETS JPEG::JPEG)
else()
    message(STATUS "CImgPlugin: JPEG Image Format disabled")
endif()
if(PNG_FOUND)
    message(STATUS "CImgPlugin: PNG Image Format enabled")
    list(APPEND CIMG_CFLAGS -Dcimg_use_png)
    if(UNIX)
        # Prevent pngconf error (see https://bugs.launchpad.net/ubuntu/+source/libpng/+bug/218409)
        list(APPEND CIMG_CFLAGS -DPNG_SKIP_SETJMP_CHECK)
    endif()
    list(APPEND DEP_TARGETS PNG::PNG)
else()
    message(STATUS "CImgPlugin: PNG Image Format disabled")
endif()
if(ZLIB_FOUND)
    list(APPEND DEP_TARGETS ZLIB::ZLIB)
endif()

add_library(${PROJECT_NAME} SHARED ${HEADER_FILES} ${SOURCE_FILES})

target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC "$<BUILD_INTERFACE:${CImg_INCLUDE_DIRS}>")
target_compile_options(${PROJECT_NAME} PUBLIC ${CIMG_CFLAGS})
target_link_libraries(${PROJECT_NAME} PUBLIC Sofa.Helper Sofa.DefaultType )
target_link_libraries(${PROJECT_NAME} INTERFACE CImg)
target_link_libraries(${PROJECT_NAME} PUBLIC ${DEP_TARGETS})

if(SOFA_BUILD_RELEASE_PACKAGE OR CMAKE_SYSTEM_NAME STREQUAL Windows)
    sofa_install_libraries(TARGETS ${DEP_TARGETS} ${DEP_ZSTD})
endif()

if(MSVC)
    target_link_options(${PROJECT_NAME} PRIVATE "/DEF:${CMAKE_CURRENT_SOURCE_DIR}/exports.def")
endif()

## Install rules for the library; CMake package configurations files
sofa_create_package_with_targets(
    PACKAGE_NAME ${PROJECT_NAME}
    PACKAGE_VERSION ${PROJECT_VERSION}
    TARGETS ${PROJECT_NAME} CImg AUTO_SET_TARGET_PROPERTIES 
    INCLUDE_SOURCE_DIR "src"
    INCLUDE_INSTALL_DIR "CImgPlugin"
    RELOCATABLE "plugins"
    )

install(FILES
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindCImg.cmake"
    DESTINATION lib/cmake/Modules
    COMPONENT headers
    )

# Tests
# If SOFA_BUILD_TESTS exists and is OFF, then these tests will be auto-disabled
cmake_dependent_option(CIMGPLUGIN_BUILD_TESTS "Compile the automatic tests" ON "SOFA_BUILD_TESTS OR NOT DEFINED SOFA_BUILD_TESTS" OFF)
if(CIMGPLUGIN_BUILD_TESTS)
    enable_testing()
    add_subdirectory(CImgPlugin_test)
endif()
