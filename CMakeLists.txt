cmake_minimum_required (VERSION 2.6)
project(openDTAM)

set(CMAKE_BUILD_TYPE Debug) 
cmake_policy(SET CMP0012 NEW)

# ROS setup
include($ENV{ROS_ROOT}/core/rosbuild/rosbuild.cmake)
rosbuild_init()
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_SOURCE_DIR}/build)
set(LIBRARY_OUTPUT_PATH ${CMAKE_SOURCE_DIR}/build)

# add dynamic reconfigure api
rosbuild_find_ros_package(dynamic_reconfigure)
include(${dynamic_reconfigure_PACKAGE_PATH}/cmake/cfgbuild.cmake)
gencfg()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -g -ggdb ")

find_package(OpenCV 2.4.9 EXACT REQUIRED)
find_package(CUDA REQUIRED)
find_package(Boost COMPONENTS system thread)
find_package(PCL 1.7)

###############################################################################
macro (add_sources)
    file (RELATIVE_PATH _relPath "${CMAKE_SOURCE_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}")
    foreach (_src ${ARGN})
        if (_relPath)
            list (APPEND DTAM_SOURCES "${_relPath}/${_src}")
        else()
            list (APPEND DTAM_SOURCES "${_src}")
        endif()
    endforeach()
    if (_relPath)
        # propagate SRCS to parent directory
        set (DTAM_SOURCES ${DTAM_SOURCES} PARENT_SCOPE)
    endif()
endmacro(add_sources)
###############################################################################

set(BASEPATH "${CMAKE_SOURCE_DIR}/src")

add_subdirectory(src/dense-mapping)

include_directories("${BASEPATH}" ${OpenCV_INCLUDE_DIRS} ${CUDA_INCLUDE_DIRS} ${Boost_INCLUDE_DIRS}) # ${PCL_INCLUDE_DIRS})

include_directories(${PCL_INCLUDE_DIRS})
link_directories(${PCL_LIBRARY_DIRS})
add_definitions(${PCL_DEFINITIONS})

# Future proofing gpu code by forcing device independent PTX code to be generated, instead of gpu specific cubin code. See:
# https://devtalk.nvidia.com/default/topic/938165/when-will-cuda-8-be-released-1080-can-t-run-with-7-5/
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_30,code=compute_30; -gencode arch=compute_35,code=compute_35;--use_fast_math;--restrict; -lineinfo; -g; -G; -O3)
cuda_add_library(openDTAM_LIB SHARED ${DTAM_SOURCES})
target_link_libraries(openDTAM_LIB ${OpenCV_LIBS} ${Boost_LIBRARIES})

rosbuild_add_executable(dense-mapping-orb-slam-tracking src/dense-mapping/DenseMapper.cpp src/dense-mapping/dense-mapping-orbslam-tracking.cpp)
target_link_libraries(dense-mapping-orb-slam-tracking ${OpenCV_LIBS} openDTAM_LIB ${Boost_LIBRARIES} ${PCL_LIBRARIES})
