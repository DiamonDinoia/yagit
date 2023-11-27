# This file is part of 'yet Another Gamma Index Tool'.
#
# 'yet Another Gamma Index Tool' is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# 'yet Another Gamma Index Tool' is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with 'yet Another Gamma Index Tool'; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301  USA

cmake_minimum_required(VERSION 3.8)
project(gi_core)

set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(common)

if (NOT CPM_DOWNLOAD_VERSION)
    set(CPM_DOWNLOAD_VERSION 0.38.5)
endif ()

if (CPM_SOURCE_CACHE)
    set(CPM_DOWNLOAD_LOCATION "${CPM_SOURCE_CACHE}/cpm/CPM_${CPM_DOWNLOAD_VERSION}.cmake")
elseif (DEFINED ENV{CPM_SOURCE_CACHE})
    set(CPM_DOWNLOAD_LOCATION "$ENV{CPM_SOURCE_CACHE}/cpm/CPM_${CPM_DOWNLOAD_VERSION}.cmake")
else ()
    set(CPM_DOWNLOAD_LOCATION "${CMAKE_BINARY_DIR}/cmake/CPM_${CPM_DOWNLOAD_VERSION}.cmake")
endif ()
if (NOT (EXISTS ${CPM_DOWNLOAD_LOCATION}))
    message(STATUS "Downloading CPM.cmake to ${CPM_DOWNLOAD_LOCATION}")
    file(DOWNLOAD
            https://github.com/TheLartians/CPM.cmake/releases/download/v${CPM_DOWNLOAD_VERSION}/CPM.cmake
            ${CPM_DOWNLOAD_LOCATION}
    )
endif ()
include(${CPM_DOWNLOAD_LOCATION})


# GI_CORE
find_package(OpenMP)

CPMFindPackage(
  NAME libsimdpp
  GITHUB_REPOSITORY TheTryton/libsimdpp
  VERSION 2.1.1
)

CPMFindPackage(
  NAME vexcl
  GITHUB_REPOSITORY ddemidov/vexcl
  GIT_TAG 1.4.2
)

find_c_and_cpp_files("${CMAKE_CURRENT_SOURCE_DIR}/include" gi_core_headers)
find_c_and_cpp_files("${CMAKE_CURRENT_SOURCE_DIR}/src" gi_core_sources)
find_c_and_cpp_files("${CMAKE_CURRENT_SOURCE_DIR}/compat" gi_core_compatibility_sources)

set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${libsimdpp_SOURCE_DIR}/cmake")
include(SimdppMultiarch)
simdpp_get_compilable_archs(compilable_archs)
simdpp_multiarch(generated_arch_files_d_ffaa "dispatch/data/allocation/fast_float_aligned_allocator_impl.cpp" ${compilable_archs})
simdpp_multiarch(generated_arch_files_v "dispatch/math/vectorized/gamma_index_range_impl.cpp" ${compilable_archs})
simdpp_multiarch(generated_arch_files_v_omp "dispatch/math/vectorized/openmp/gamma_index_range_impl.cpp" ${compilable_archs})

source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR} FILES ${gi_core_headers} ${gi_core_sources} ${gi_core_compatibility_sources})

add_library(gi_core STATIC
        ${gi_core_headers}
        ${gi_core_sources}
        ${generated_arch_files_d_ffaa}
        ${generated_arch_files_v}
        ${generated_arch_files_v_omp}
        ${gi_core_compatibility_sources})
#target_link_libraries(gi_core PUBLIC VexCL::OpenCL) not linking as for the moment GPU support is not developed
target_include_directories(gi_core PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include ${CMAKE_CURRENT_SOURCE_DIR}/compat ${libsimdpp_SOURCE_DIR})
target_include_directories(gi_core PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
if (OPENMP_FOUND)
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
    target_compile_definitions(gi_core PUBLIC -DYAGIT_OPENMP)
else()
	message(WARNING "Selected compiler doesn't support OPENMP!")
endif()

target_compile_definitions(gi_core PUBLIC -DVEXCL_SHOW_KERNELS)

set_target_properties(gi_core
    PROPERTIES
    CXX_STANDARD 20
    CXX_STANDARD_REQUIRED YES
    CXX_EXTENSIONS NO
)

set_default_output_directories(gi_core)

# GI_CORE_TESTS
if (YAGIT_ENABLE_TESTS)
    find_package(Boost COMPONENTS system filesystem unit_test_framework REQUIRED)
    add_subdirectory(tests)
endif ()
if (YAGIT_ENABLE_BENCHMARKS)
    add_subdirectory(performance_analysis)
endif ()
# GI_PERFORMANCE_ANALYSIS
