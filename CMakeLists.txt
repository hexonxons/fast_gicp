cmake_minimum_required(VERSION 3.10.0)
project(fast_gicp)

option(BUILD_apps "Build application programs" ON)
option(BUILD_test "Build test programs" OFF)
option(BUILD_PYTHON_BINDINGS "Build python bindings" OFF)

if(${CMAKE_SYSTEM_PROCESSOR} STREQUAL "aarch64")
else()
  add_definitions(-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2)
  set(CMAKE_C_FLAGS "-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2")
  set(CMAKE_CXX_FLAGS "-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2")
endif()

set(CMAKE_BUILD_TYPE "Release")

find_package(PCL REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(ament_cmake REQUIRED)
find_package(OpenMP)
find_package(CUDA REQUIRED)

###########
## Build ##
###########

add_library(${PROJECT_NAME} SHARED
  src/fast_gicp/gicp/lsq_registration.cpp
  src/fast_gicp/gicp/fast_gicp.cpp
  src/fast_gicp/gicp/fast_gicp_st.cpp
  src/fast_gicp/gicp/fast_vgicp.cpp
)
target_link_libraries(${PROJECT_NAME}
  ${PCL_LIBRARIES}
  OpenMP::OpenMP_CXX
  Eigen3::Eigen
)
target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
  ${PCL_INCLUDE_DIRS}
)

### APPS ###
if(BUILD_apps)
  add_executable(gicp_align src/align.cpp)
  add_dependencies(gicp_align ${PROJECT_NAME})
  target_link_libraries(gicp_align
    ${PCL_LIBRARIES}
    ${PROJECT_NAME}
  )

  add_executable(gicp_kitti src/kitti.cpp)
  add_dependencies(gicp_kitti ${PROJECT_NAME})
  target_link_libraries(gicp_kitti
    ${PCL_LIBRARIES}
    ${PROJECT_NAME}
  )
endif()

### Python bindings ###
if(BUILD_PYTHON_BINDINGS)
  add_subdirectory(thirdparty/pybind11)
  pybind11_add_module(pygicp
    src/python/main.cpp
  )
  target_include_directories(pygicp PUBLIC
    include
    ${PCL_INCLUDE_DIRS}
  )
  target_link_libraries(pygicp PRIVATE
    ${PROJECT_NAME}
    Eigen3::Eigen
  )
endif()

### CUDA ###
set(CUDA_NVCC_FLAGS "--expt-relaxed-constexpr")
add_definitions(-DUSE_VGICP_CUDA)

cuda_add_library(fast_vgicp_cuda SHARED
  src/fast_gicp/cuda/fast_vgicp_cuda.cu
  src/fast_gicp/cuda/brute_force_knn.cu
  src/fast_gicp/cuda/covariance_estimation.cu
  src/fast_gicp/cuda/covariance_estimation_rbf.cu
  src/fast_gicp/cuda/covariance_regularization.cu
  src/fast_gicp/cuda/gaussian_voxelmap.cu
  src/fast_gicp/cuda/find_voxel_correspondences.cu
  src/fast_gicp/cuda/compute_derivatives.cu
  src/fast_gicp/cuda/compute_mahalanobis.cu
  src/fast_gicp/cuda/ndt_cuda.cu
  src/fast_gicp/cuda/ndt_compute_derivatives.cu
)

target_include_directories(fast_vgicp_cuda PRIVATE
  include
  thirdparty/nvbio
  ${CUDA_INCLUDE_DIRS}
)
target_link_libraries(fast_vgicp_cuda
  ${CUDA_LIBRARY_DIRS}
  Eigen3::Eigen
)
cuda_add_cublas_to_target(fast_vgicp_cuda)

# add vgicp_cuda to libfast_gicp
target_sources(${PROJECT_NAME} PRIVATE
  src/fast_gicp/gicp/fast_vgicp_cuda.cpp
  src/fast_gicp/ndt/ndt_cuda.cpp
)

target_link_libraries(${PROJECT_NAME}
  fast_vgicp_cuda
)
add_dependencies(${PROJECT_NAME} fast_vgicp_cuda)

### TEST ###
if(BUILD_test)
  find_package(GTest REQUIRED)

  add_executable(gicp_test src/test/gicp_test.cpp)
  add_dependencies(gicp_test ${PROJECT_NAME})
  target_link_libraries(gicp_test ${GTEST_LIBRARIES} ${PCL_LIBRARIES} ${PROJECT_NAME})
  gtest_add_tests(TARGET gicp_test WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} EXTRA_ARGS "${CMAKE_SOURCE_DIR}/data")
endif()

install(
  TARGETS ${PROJECT_NAME} fast_vgicp_cuda
  EXPORT export_${PROJECT_NAME}
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin)

ament_target_dependencies(${PROJECT_NAME}
  Eigen3
  PCL
)
ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)

ament_package()
