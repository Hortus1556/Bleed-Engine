cmake_minimum_required(VERSION 3.20)
project(BleedEngine CXX C)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(SDL3 REQUIRED CONFIG)

# lunasvg (MIT): rasterises vector art (SVG) at the display's resolution.
set(BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
add_subdirectory(third_party/lunasvg-3.5.0 EXCLUDE_FROM_ALL)

# libtess2 (SGI Free Software License B 2.0): cuts vector art's filled
# outlines into triangles for the GPU (flash/VectorMesh).
file(GLOB TESS_SOURCES third_party/libtess2/Source/*.c)
add_library(tess2 STATIC ${TESS_SOURCES})
target_include_directories(tess2 PUBLIC third_party/libtess2/Include)
target_compile_options(tess2 PRIVATE -w)

# Bleed Engine itself: the core, the platform layer, the engine (cocos and
# UIKit subsets) and the Flash runtime. src/surgeon is deliberately not
# part of this library — it's meant to be compiled directly into a game's
# own executable alongside BleedEngine, the same way upstream does it.
file(GLOB_RECURSE ENGINE_SOURCES CONFIGURE_DEPENDS
  src/core/*.cpp src/platform/*.cpp src/engine/*.cpp src/flash/*.cpp)
add_library(BleedEngine STATIC ${ENGINE_SOURCES})
target_include_directories(BleedEngine PUBLIC src third_party)
if(WIN32)
  target_link_libraries(BleedEngine PUBLIC SDL3::SDL3 opengl32 mfplat mfreadwrite mfuuid ole32 shell32 lunasvg::lunasvg plutovg::plutovg tess2)
else()
  target_link_libraries(BleedEngine PUBLIC SDL3::SDL3 lunasvg::lunasvg plutovg::plutovg tess2)
endif()
target_compile_options(BleedEngine PUBLIC -Wall -Wno-unused-parameter)

add_executable(extract_swf src/tools/extract_swf.cpp)
target_link_libraries(extract_swf PRIVATE BleedEngine)

add_executable(svgrender src/tools/svgrender.cpp)
target_include_directories(svgrender PRIVATE third_party)
target_link_libraries(svgrender PRIVATE lunasvg::lunasvg)
