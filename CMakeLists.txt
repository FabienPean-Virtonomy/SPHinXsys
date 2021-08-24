#---------------------------------------
# SPHINXSYS V.1.0.0
#---------------------------------------
cmake_minimum_required(VERSION 3.16)
project(SPHINXsys VERSION 1.0.0 LANGUAGES CXX)

set(SPHINXSYS_PROJECT_DIR ${PROJECT_SOURCE_DIR})
if (EMSCRIPTEN)
   if(CCACHE_FOUND)
       set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
   endif()

   add_definitions(-D_LIBCPP_NO_EXPERIMENTAL_DEPRECATION_WARNING_FILESYSTEM)
endif()
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${SPHINXSYS_PROJECT_DIR}/cmake)

###### Change settings in this file if you want to build with Simbody and/or oneTBB source code ######
option(ONLY_3D "ONLY_3D" 0)
include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/Common.cmake")
include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/Dependency_free_settings.cmake")
if (EMSCRIPTEN)
    set(BUILD_WITH_ONETBB 0)
    remove_definitions(-DTBB_2021_2_0)
    set(TBB_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/wasmtbb)
    set(EIGEN_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/eigen)

    INCLUDE_DIRECTORIES("${TBB_ROOT}/include")
    INCLUDE_DIRECTORIES("${EIGEN_ROOT}")

    set(OBJDIR "${TBB_ROOT}/build/linux_wasm32_emscripten_emscripten_wasm32_release/")
    FILE(GLOB TBB_OBJECTS ${OBJDIR}/*.o)
endif()

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
    # Set the possible values of build type for cmake-gui
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
            "MinSizeRel" "RelWithDebInfo")
endif()

set(ACTIVATE_CUDA OFF CACHE BOOL "Activate CUDA?")
if(ACTIVATE_CUDA)
    include(CheckLanguage)
    check_language(CUDA)
    if(CMAKE_CUDA_COMPILER)
      enable_language(CUDA)
    else()
      message(STATUS "No CUDA support!")
    endif()

    set(BUILD_GPU_ARCH "75" CACHE STRING "Define the CUDA architecture")
    #option(BUILD_GPU_FORCE_INLINES "CUDA 7.5 AND Ubuntu 16.04 needs this enabled" OFF)
    #string(APPEND CMAKE_CUDA_FLAGS " -arch=sm_${BUILD_GPU_ARCH} UAMMD_DEBUG")
    string(APPEND CMAKE_CUDA_FLAGS " -arch=sm_${BUILD_GPU_ARCH}")

endif()

if(MSVC)
    SET(CMAKE_CL_64 1)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}" " /MP")
    add_definitions(-D_USE_MATH_DEFINES)
elseif(DEFINED EMSCRIPTEN)
    set(PRE_POST "")
    #-s ALLOW_MEMORY_GROWTH=1
    set(EM_FLAGS "${PRE_POST} -fPIC -g -O0 --profiling -DUNIX -D__linux__=1 --bind -std=c++11 -s USE_PTHREADS=1 -s DISABLE_EXCEPTION_CATCHING=0")
    set(EM_FLAGS "${PRE_POST} -fPIC -g -O0 --profiling -DUNIX -D__linux__=1 --bind -std=c++11 -s USE_PTHREADS=0 -s DISABLE_EXCEPTION_CATCHING=0")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${EM_FLAGS}")
    set(EM_FLAGS " -s INITIAL_MEMORY=536870912 -s ASSERTIONS=1 -s ALLOW_MEMORY_GROWTH=1 -s ERROR_ON_UNDEFINED_SYMBOLS=0")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${EM_FLAGS}")
else(MSVC)
    add_compile_definitions(__linux__=1)
    set(CMAKE_CXX_FLAGS "-Wall -Werror -std=c++11 -DUNIX -pthread")
    set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG")
    set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -ggdb")
    IF(${CMAKE_BUILD_TYPE} MATCHES "Debug")
        set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS}")
    ELSE(${CMAKE_BUILD_TYPE} MATCHES "Debug")
        set(CMAKE_CXX_FLAGS "-fPIC ${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_RELEASE}")
    ENDIF(${CMAKE_BUILD_TYPE} MATCHES "Debug")

endif(MSVC)
add_compile_definitions(TBB_SUPPRESS_DEPRECATED_MESSAGES=1)

message("===========================================")
message("${PROJECT_NAME}, VERSION ${PROJECT_VERSION}")
message("Cmake build type: ${CMAKE_BUILD_TYPE}")
message("CXX flags: ${CMAKE_CXX_FLAGS}")
message("CUDA flags: ${CMAKE_CUDA_FLAGS}")
message("===========================================")

########### GPU - compliler flags for physical properties #############
# 1. Turn ON if the body force is time dependent, e.g. sloshing tank 
option(_TIMEDEPENDENT_BODYFORCE_ "Enable time dependent body force" OFF)
if (${_TIMEDEPENDENT_BODYFORCE_})
    add_definitions(-D_TIMEDEPENDENT_BODYFORCE_)
endif()
#######################################################################

enable_testing()

if(NOT WASM_BUILD)
    include(Common)
    include(Import3rdpartylibs)
endif()

option(BUILD_TESTS "BUILD_TESTS" 1)
add_subdirectory(SPHINXsys)

if(BUILD_TESTS)
    add_subdirectory(tests)
endif()

add_subdirectory(cases_user)

if(ACTIVATE_CUDA)
    add_subdirectory(cases_test_gpu)
endif()
