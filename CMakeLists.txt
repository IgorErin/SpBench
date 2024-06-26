cmake_minimum_required(VERSION 3.15)
project(SpBench LANGUAGES CXX CUDA)

# Exposed options. Disable something, if do not want to build cuda or cl stuff.
option(BENCH_WITH_CUBOOL      "Add cubool lib and related benchmarks" OFF)
option(BENCH_WITH_CUSP        "Add cusp lib and related benchmarks" OFF)
option(BENCH_WITH_CUSPARSE    "Add cuSPARSE lib and related benchmarks" OFF)
option(BENCH_WITH_CLSPARSE    "Add clSPARSE lib and related benchmarks" OFF)
option(BENCH_WITH_CLBOOL      "Add clbool lib and related benchmarks" OFF)
option(BENCH_WITH_SUITESPARSE "Add GraphBLAS:SuiteSparse lib and related benchmarks" ON)

add_library(sp_bench_base INTERFACE)
target_include_directories(sp_bench_base INTERFACE ${CMAKE_CURRENT_LIST_DIR}/src)

# Append here all benchmark targets
set(TARGETS)

# Cubool specific stuff
if (BENCH_WITH_CUBOOL)
    set(CUBOOL_WITH_CUDA ON CACHE BOOL "" FORCE)
    set(CUBOOL_WITH_SEQUENTIAL ON CACHE BOOL "" FORCE)
    set(CUBOOL_BUILD_TESTS OFF CACHE BOOL "" FORCE)
    set(CUBOOL_COPY_TO_PY_PACKAGE OFF CACHE BOOL "" FORCE)
    add_subdirectory(thirdparty/cubool)

    add_executable(cubool_prepare_data src/cubool_prepare_data.cpp)
    target_link_libraries(cubool_prepare_data PUBLIC cubool)
    target_link_libraries(cubool_prepare_data PUBLIC sp_bench_base)

    set(CUBOOl_TARGETS)
    add_executable(cubool_mult src/cubool_multiply.cpp)
    add_executable(cubool_add src/cubool_add.cpp)
    list(APPEND CUBOOl_TARGETS cubool_mult cubool_add)

    foreach(CUBOOL_TARGET ${CUBOOl_TARGETS})
        target_link_libraries(${CUBOOL_TARGET} PUBLIC cubool)
        target_link_libraries(${CUBOOL_TARGET} PUBLIC sp_bench_base)

        list(APPEND TARGETS ${CUBOOL_TARGET})
    endforeach()
endif()

# CUSP specifics
if (BENCH_WITH_CUSP)
    add_library(cusp INTERFACE)
    target_include_directories(cusp INTERFACE ${CMAKE_CURRENT_LIST_DIR}/thirdparty/cusp)

    set(CUSP_TARGETS)

    add_executable(cusp_basic src/cusp_basic.cu)
    add_executable(cusp_mult src/cusp_multiply.cu)
    add_executable(cusp_add src/cusp_add.cu)
    add_executable(cusp_mult_add src/cusp_multiply_add.cu)
    list(APPEND CUSP_TARGETS cusp_basic cusp_mult cusp_add cusp_mult_add)

    foreach(CUSP_TARGET ${CUSP_TARGETS})
        target_link_libraries(${CUSP_TARGET} PUBLIC cusp)
        target_link_libraries(${CUSP_TARGET} PUBLIC sp_bench_base)

        target_compile_features(${CUSP_TARGET} PUBLIC cxx_std_14)

        set_target_properties(${CUSP_TARGET} PROPERTIES CXX_STANDARD 17)
        set_target_properties(${CUSP_TARGET} PROPERTIES CXX_STANDARD_REQUIRED ON)
        set_target_properties(${CUSP_TARGET} PROPERTIES CUDA_STANDARD 14)
        set_target_properties(${CUSP_TARGET} PROPERTIES CUDA_STANDARD_REQUIRED ON)
        set_target_properties(${CUSP_TARGET} PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

        list(APPEND TARGETS ${CUSP_TARGET})
    endforeach()
endif()

if (BENCH_WITH_CUSPARSE)
    set(CUSPARSE_TARGETS)

    find_package(CUDA REQUIRED)

    add_executable(cusparse_mult src/cusparse_multiply.cu)
    add_executable(cusparse_add src/cusparse_add.cu)
    list(APPEND CUSPARSE_TARGETS cusparse_add cusparse_mult)

    foreach(CUSPARSE_TARGET ${CUSPARSE_TARGETS})
        target_link_libraries(${CUSPARSE_TARGET} PUBLIC sp_bench_base)
        target_link_libraries(${CUSPARSE_TARGET} PUBLIC cusparse)

        target_compile_features(${CUSPARSE_TARGET} PUBLIC cxx_std_14)
        set_target_properties(${CUSPARSE_TARGET} PROPERTIES CXX_STANDARD 17)
        set_target_properties(${CUSPARSE_TARGET} PROPERTIES CXX_STANDARD_REQUIRED ON)
        set_target_properties(${CUSPARSE_TARGET} PROPERTIES CUDA_STANDARD 14)
        set_target_properties(${CUSPARSE_TARGET} PROPERTIES CUDA_STANDARD_REQUIRED ON)
        set_target_properties(${CUSPARSE_TARGET} PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

        list(APPEND TARGETS ${CUSP_TARGET})
    endforeach()
endif()

if (BENCH_WITH_CLSPARSE)
    set(BUILD_clSPARSE ON CACHE BOOL "" FORCE)
    set(BUILD_Boost ON CACHE BOOL "" FORCE)
    set(BUILD_gMock ON CACHE BOOL "" FORCE)
    set(BUILD_MTX ON CACHE BOOL "" FORCE)
    set(BUILD_SAMPLES OFF CACHE BOOL "" FORCE)
    set(USE_SYSTEM_CL2HPP OFF CACHE BOOL "" FORCE)
    add_subdirectory(thirdparty/clSPARSE)

    set(CLSPARSE_TARGETS)

    add_executable(clsparse_basic src/clsparse_basic.cpp)
    add_executable(clsparse_mult src/clsparse_multiply.cpp)
    list(APPEND CLSPARSE_TARGETS clsparse_basic clsparse_mult)

    set(clSPARSE_PATH ${CMAKE_BINARY_DIR}/thirdparty/clSPARSE/clSPARSE-build)
    message(STATUS "${clSPARSE_PATH}/library")

    find_package(OpenCL REQUIRED)

    if (CMAKE_BUILD_TYPE STREQUAL "Debug")
        set(clSPARSE_LIB_NAME clSPARSEd)
    elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
        set(clSPARSE_LIB_NAME clSPARSE)
    else()
        message(SEND_ERROR "Unknown build type")
    endif()

    foreach(CLSPARSE_TARGET ${CLSPARSE_TARGETS})
        add_dependencies(${CLSPARSE_TARGET} clSPARSE)
        target_link_libraries(${CLSPARSE_TARGET} PUBLIC sp_bench_base)
        target_link_libraries(${CLSPARSE_TARGET} PUBLIC OpenCL)
        target_link_directories(${CLSPARSE_TARGET} PUBLIC ${clSPARSE_PATH}/library)
        target_include_directories(${CLSPARSE_TARGET} PUBLIC thirdparty/clSPARSE/src/include)
        target_include_directories(${CLSPARSE_TARGET} PUBLIC ${clSPARSE_PATH}/library)
        # Use link libraries command to provide extra libs in correct order (proposed by Nikita)
        target_link_libraries(${CLSPARSE_TARGET} PUBLIC "-l${clSPARSE_LIB_NAME}")

        target_compile_features(${CLSPARSE_TARGET} PUBLIC cxx_std_11)

        set_target_properties(${CLSPARSE_TARGET} PROPERTIES CXX_STANDARD 11)
        set_target_properties(${CLSPARSE_TARGET} PROPERTIES CXX_STANDARD_REQUIRED ON)

        list(APPEND TARGETS ${CLSPARSE_TARGET})
    endforeach()
endif()

if (BENCH_WITH_CLBOOL)
    add_subdirectory(thirdparty/clbool)

    set(CLBOOL_TARGETS)

    add_executable(clbool_mult_hash src/clbool_multiply_hash.cpp)
    add_executable(clbool_mult src/clbool_multiply.cpp)
    add_executable(clbool_add src/clbool_add.cpp)
    list(APPEND CLBOOL_TARGETS clbool_mult_hash clbool_mult clbool_add)

    foreach(CLBOOL_TARGET ${CLBOOL_TARGETS})
        target_link_libraries(${CLBOOL_TARGET} PUBLIC sp_bench_base)
        target_link_libraries(${CLBOOL_TARGET} PUBLIC clbool)
        target_compile_features(${CLBOOL_TARGET} PUBLIC cxx_std_14)

        set_target_properties(${CLBOOL_TARGET} PROPERTIES CXX_STANDARD 17)
        set_target_properties(${CLBOOL_TARGET} PROPERTIES CXX_STANDARD_REQUIRED ON)

        list(APPEND TARGETS ${CLBOOL_TARGET})
    endforeach()
endif()

if (BENCH_WITH_SUITESPARSE)
    set(SUITESPARSE_TARGETS)

    add_executable(suitesparse_mult src/suitesparse_multiply.cpp)
    add_executable(suitesparse_mult_any_pair src/suitesparse_multiply_any_pair.cpp)
    add_executable(suitesparse_add src/suitesparse_add.cpp)
    add_executable(suitesparse_add_any_pair src/suitesparse_add_any_pair.cpp)
    list(APPEND SUITESPARSE_TARGETS suitesparse_mult suitesparse_mult_any_pair suitesparse_add suitesparse_add_any_pair)

    foreach(SUITESPARSE_TARGET ${SUITESPARSE_TARGETS})
        target_link_libraries(${SUITESPARSE_TARGET} PUBLIC sp_bench_base)
        target_link_libraries(${SUITESPARSE_TARGET} PUBLIC graphblas)

        # The graphblas is installed here, must explicitly add path to be sure, that link will be successful
        target_link_directories(${SUITESPARSE_TARGET} PUBLIC /usr/local/lib)
        target_compile_features(${SUITESPARSE_TARGET} PUBLIC cxx_std_14)
        set_target_properties(${SUITESPARSE_TARGET} PROPERTIES CXX_STANDARD 17)
        set_target_properties(${SUITESPARSE_TARGET} PROPERTIES CXX_STANDARD_REQUIRED ON)

        list(APPEND TARGETS ${SUITESPARSE_TARGET})
    endforeach()
endif()

# Some fancy stuff here
foreach(TARGET ${TARGETS})
    message(STATUS "Build target benchmark ${TARGET}")
endforeach()

# Copy data into build directory
file(COPY ${CMAKE_CURRENT_LIST_DIR}/data/ DESTINATION ${CMAKE_BINARY_DIR}/data/)

# Copy scripts into build directory
file(GLOB SCRIPTS "*.sh")
file(COPY ${SCRIPTS} DESTINATION ${CMAKE_BINARY_DIR})

# Unpack data for benchmarks into build directory
add_custom_target(unzip_dataset ALL)
set(DATASET_ARCHIVES dataset.zip)

foreach(ARCHIVE_NAME ${DATASET_ARCHIVES})
    add_custom_command(TARGET unzip_dataset PRE_BUILD
            COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_LIST_DIR}/dataset/${ARCHIVE_NAME}
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/data
            DEPENDS ${CMAKE_SOURCE_DIR}/dataset/${ARCHIVE_NAME}
            COMMENT "Unpacking benchmark dataset from ${ARCHIVE_NAME}"
            VERBATIM)
endforeach()