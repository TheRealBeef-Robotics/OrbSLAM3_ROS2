cmake_minimum_required(VERSION 3.5)
project(orb_slam3)

IF(NOT CMAKE_BUILD_TYPE)
  SET(CMAKE_BUILD_TYPE Release)
ENDIF()

MESSAGE("Build type: " ${CMAKE_BUILD_TYPE})

option(ENABLE_VIEWER "enable render viewer" OFF)
MESSAGE("ENABLE_VIEWER: ${ENABLE_VIEWER}")
if (ENABLE_VIEWER)
    add_definitions(-DENABLE_VIEWER)
    find_package(Pangolin REQUIRED)
endif()

option(SUPPORT_SUPERPOINT "support Superpoint network instead of ORBextractor" OFF)
set(SUPERPOINTFILE "")
if (SUPPORT_SUPERPOINT)
    add_definitions(-DSUPPORT_SUPERPOINT)
    set(SUPERPOINTFILE src/SuperPointExtractor.cc)
    set(CMAKE_PREFIX_PATH "../../../../../install")
#    find_package(dnn_node REQUIRED)
#    message("dnn_node_includes: ${dnn_node_INCLUDE_DIR}")
#    message("dnn_node_includes: ${dnn_node_INCLUDE_DIRS}")
    include_directories(
            ${CMAKE_SYSROOT}/usr/include/
#            ${dnn_node_INCLUDE_DIRS}
    )
    link_directories(
            ${CMAKE_SYSROOT}/usr/lib/hbbpu/
    )
endif()


find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(rclcpp_components REQUIRED)
find_package(cv_bridge REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)

set(CMAKE_PREFIX_PATH "${PROJECT_SOURCE_DIR}/build/;${CMAKE_PREFIX_PATH}")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -w -O3 ")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -w -O3 ")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ")
set(OpenSSL_LIBRARY_PATH ${CMAKE_SYSROOT}/usr/lib/aarch64-linux-gnu/)


# Check C++11 or C++0x support
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++17" COMPILER_SUPPORTS_CXX17)
CHECK_CXX_COMPILER_FLAG("-std=c++14" COMPILER_SUPPORTS_CXX14)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX17)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")
    add_definitions(-DCOMPILEDWITHC11)
    message(STATUS "Using flag -std=c++17.")
elseif(COMPILER_SUPPORTS_CXX14)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
    add_definitions(-DCOMPILEDWITHC11)
    message(STATUS "Using flag -std=c++14.")
elseif(COMPILER_SUPPORTS_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    add_definitions(-DCOMPILEDWITHC11)
    message(STATUS "Using flag -std=c++11.")
elseif(COMPILER_SUPPORTS_CXX0x)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
    add_definitions(-DCOMPILEDWITHC0X)
    message(STATUS "Using flag -std=c++0x.")
else()
    message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules)


find_package(OpenCV)
   if(NOT OpenCV_FOUND)
      message(FATAL_ERROR "OpenCV > 4.4 not found.")
   endif()

MESSAGE("OPENCV VERSION:")
MESSAGE(${OpenCV_VERSION})

find_package(Eigen3 REQUIRED)
find_package(realsense2)

include_directories(
${PROJECT_SOURCE_DIR}
${PROJECT_SOURCE_DIR}/include
${PROJECT_SOURCE_DIR}/include/CameraModels
${PROJECT_SOURCE_DIR}/Thirdparty/Sophus
${EIGEN3_INCLUDE_DIR}
${Pangolin_INCLUDE_DIRS}
${TORCH_INCLUDE_DIRS}
)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)

add_library(${PROJECT_NAME} SHARED
src/System.cc
src/Tracking.cc
src/LocalMapping.cc
src/LoopClosing.cc
src/ORBextractor.cc
src/ORBmatcher.cc
${SUPERPOINTFILE}
src/FrameDrawer.cc
src/Converter.cc
src/MapPoint.cc
src/KeyFrame.cc
src/Atlas.cc
src/Map.cc
src/MapDrawer.cc
src/Optimizer.cc
src/Frame.cc
src/KeyFrameDatabase.cc
src/Sim3Solver.cc
src/Viewer.cc
src/ImuTypes.cc
src/G2oTypes.cc
src/CameraModels/Pinhole.cpp
src/CameraModels/KannalaBrandt8.cpp
src/OptimizableTypes.cpp
src/MLPnPsolver.cpp
src/GeometricTools.cc
src/TwoViewReconstruction.cc
src/Config.cc
src/Settings.cc
src/threadpool.cpp
include/System.h
include/Tracking.h
include/LocalMapping.h
include/LoopClosing.h
include/ORBextractor.h
include/ORBmatcher.h
include/FrameDrawer.h
include/Converter.h
include/MapPoint.h
include/KeyFrame.h
include/Atlas.h
include/Map.h
include/MapDrawer.h
include/Optimizer.h
include/Frame.h
include/KeyFrameDatabase.h
include/Sim3Solver.h
include/Viewer.h
include/ImuTypes.h
include/G2oTypes.h
include/CameraModels/GeometricCamera.h
include/CameraModels/Pinhole.h
include/CameraModels/KannalaBrandt8.h
include/OptimizableTypes.h
include/MLPnPsolver.h
include/GeometricTools.h
include/TwoViewReconstruction.h
include/SerializationUtils.h
include/Config.h
include/Settings.h)

add_subdirectory(Thirdparty/g2o)
add_subdirectory(Thirdparty/DBoW2)

message("OpenCV_LIBS: ${OpenCV_LIBS}")
message("Pangolin_LIBRARIES: ${Pangolin_LIBRARIES}")
message("TORCH_LIBRARIES: ${TORCH_LIBRARIES}")

target_link_libraries(${PROJECT_NAME}
${OpenCV_LIBS}
${EIGEN3_LIBS}
${Pangolin_LIBRARIES}
DBoW2
g2o
-lboost_serialization
#-lcrypto
#${OpenSSL_LIBRARY_PATH}/libcrypto.so.1.1
-lpthread
${TORCH_LIBRARIES}
)
ament_target_dependencies(${PROJECT_NAME}
        rclcpp
#        dnn_node
        sensor_msgs
        nav_msgs
        rclcpp_components
        cv_bridge
        tf2
        tf2_ros
	tf2_geometry_msgs)

install(DIRECTORY include/
        DESTINATION include/${PROJECT_NAME}/
        )
install(DIRECTORY Thirdparty/
        DESTINATION include/orb_slam3/Thirdparty/
        )
install(TARGETS ${PROJECT_NAME} DBoW2 g2o
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION bin/${PROJECT_NAME}
        )
install(DIRECTORY Examples
        DESTINATION share/${PROJECT_NAME}
        )
install(DIRECTORY config
        DESTINATION share/${PROJECT_NAME}
        )
install(DIRECTORY Vocabulary
        DESTINATION share/${PROJECT_NAME}
        )
ament_export_libraries(${PROJECT_NAME} DBoW2 g2o)
ament_export_include_directories(include)
ament_export_libraries(${PROJECT_NAME})
# ament_export_dependencies(dnn_node)
ament_package()
