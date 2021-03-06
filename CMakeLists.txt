# Copyright 2017, 2018 Uber Technologies, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 3.1)

set(H3_PREFIX "" CACHE STRING "Prefix for exported symbols")

# Needed due to CMP0042
set(CMAKE_MACOSX_RPATH 1)
# H3 doesn't export selected symbols right now
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
# YCM needs compilation database
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
set(CMAKE_TOOLCHAIN_FILE "${CMAKE_CURRENT_SOURCE_DIR}/cmake/toolchain.cmake"
    CACHE FILEPATH
    "Toolchain to use for building this project")

if(WIN32)
    set(SHELL PowerShell -Command)

    set(EXECUTABLE_OUTPUT_PATH bin)
    set(LIBRARY_OUTPUT_PATH bin)
else()
    set(SHELL sh -c)

    set(EXECUTABLE_OUTPUT_PATH bin)
    set(LIBRARY_OUTPUT_PATH lib)
endif()

file(READ VERSION H3_VERSION LIMIT_COUNT 1)
# Clean any newlines
string(REPLACE "\n" "" H3_VERSION "${H3_VERSION}")
set(H3_SOVERSION 1)

project(h3 VERSION ${H3_VERSION})

include(CMakeDependentOption)
include(CheckIncludeFile)
include(CTest)

include(CheckAlloca)
include(CheckVLA)

check_alloca(have_alloca)
check_vla(have_vla)

set(LIB_SOURCE_FILES
    src/h3lib/include/bbox.h
    src/h3lib/include/h3Index.h
    src/h3lib/include/h3UniEdge.h
    src/h3lib/include/geoCoord.h
    src/h3lib/include/vec2d.h
    src/h3lib/include/linkedGeo.h
    src/h3lib/include/baseCells.h
    src/h3lib/include/faceijk.h
    src/h3lib/include/vertexGraph.h
    src/h3lib/include/mathExtensions.h
    src/h3lib/include/constants.h
    src/h3lib/include/coordijk.h
    src/h3lib/include/algos.h
    src/h3lib/include/h3api.h
    src/h3lib/include/stackAlloc.h
    src/h3lib/lib/algos.c
    src/h3lib/lib/coordijk.c
    src/h3lib/lib/bbox.c
    src/h3lib/lib/h3Index.c
    src/h3lib/lib/vec2d.c
    src/h3lib/lib/linkedGeo.c
    src/h3lib/lib/geoCoord.c
    src/h3lib/lib/h3UniEdge.c
    src/h3lib/lib/mathExtensions.c
    src/h3lib/lib/vertexGraph.c
    src/h3lib/lib/faceijk.c
    src/h3lib/lib/baseCells.c)
set(APP_SOURCE_FILES
    src/apps/applib/include/test.h
    src/apps/applib/include/kml.h
    src/apps/applib/include/benchmark.h
    src/apps/applib/include/utility.h
    src/apps/applib/lib/kml.c
    src/apps/applib/lib/utility.c
    src/apps/applib/lib/test.c)
set(EXAMPLE_SOURCE_FILES
    examples/index.c
    examples/distance.c)
set(OTHER_SOURCE_FILES
    src/apps/filters/h3ToGeo.c
    src/apps/filters/h3ToComponents.c
    src/apps/filters/geoToH3.c
    src/apps/filters/h3ToGeoBoundary.c
    src/apps/filters/kRing.c
    src/apps/filters/hexRange.c
    src/apps/testapps/testVertexGraph.c
    src/apps/testapps/testCompact.c
    src/apps/testapps/testPolyfill.c
    src/apps/testapps/testKRing.c
    src/apps/testapps/testH3ToGeoBoundary.c
    src/apps/testapps/testH3ToParent.c
    src/apps/testapps/testH3Index.c
    src/apps/testapps/mkRandGeoBoundary.c
    src/apps/testapps/testGeoToH3.c
    src/apps/testapps/testH3NeighborRotations.c
    src/apps/testapps/testMaxH3ToChildrenSize.c
    src/apps/testapps/testHexRanges.c
    src/apps/testapps/testH3ToGeo.c
    src/apps/testapps/testH3ToChildren.c
    src/apps/testapps/testGeoCoord.c
    src/apps/testapps/testHexRing.c
    src/apps/testapps/testH3SetToVertexGraph.c
    src/apps/testapps/testBBox.c
    src/apps/testapps/testH3UniEdge.c
    src/apps/testapps/testLinkedGeo.c
    src/apps/testapps/mkRandGeo.c
    src/apps/testapps/testH3Api.c
    src/apps/testapps/testH3SetToLinkedGeo.c
    src/apps/miscapps/h3ToGeoBoundaryHier.c
    src/apps/miscapps/h3ToGeoHier.c
    src/apps/miscapps/generateBaseCellNeighbors.c
    src/apps/miscapps/h3ToHier.c
    src/apps/benchmarks/benchmarkPolyfill.c
    src/apps/benchmarks/benchmarkH3Api.c)

set(ALL_SOURCE_FILES
    ${LIB_SOURCE_FILES} ${APP_SOURCE_FILES} ${OTHER_SOURCE_FILES})

# Build the H3 library
add_library(h3 ${LIB_SOURCE_FILES})

find_library(M_LIB m)
if(M_LIB)
    target_link_libraries(h3 PUBLIC ${M_LIB})
endif()

if(BUILD_SHARED_LIBS)
    set_target_properties(h3 PROPERTIES SOVERSION ${H3_SOVERSION})
endif()

target_compile_definitions(h3 PUBLIC H3_PREFIX=${H3_PREFIX})
if(have_alloca)
    target_compile_definitions(h3 PUBLIC H3_HAVE_ALLOCA)
endif()
if(have_vla)
    target_compile_definitions(h3 PUBLIC H3_HAVE_VLA)
endif()
target_include_directories(h3 PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/h3lib/include>)

# Automatic code formatting
find_program(CLANG_FORMAT_PATH clang-format)
cmake_dependent_option(
    ENABLE_FORMAT "Enable running clang-format before compiling" ON
    "CLANG_FORMAT_PATH" OFF)
if(ENABLE_FORMAT)
    # Format
    add_custom_target(format
        COMMAND ${CLANG_FORMAT_PATH}
        -style=file
        -i
        ${ALL_SOURCE_FILES}
        ${EXAMPLE_SOURCE_FILES}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Formatting sources"
        )
    # Always do formatting
    add_dependencies(h3 format)
elseif(NOT CLANG_FORMAT_PATH)
    message(WARNING "clang-format was not detected, "
                    "so automatic source code reformatting is disabled")
endif()

option(ENABLE_LINTING "Run clang-tidy on source files" ON)
find_program(CLANG_TIDY_PATH "clang-tidy")
cmake_dependent_option(
    ENABLE_LINTING "Enable running clang-tidy on sources during compilation" ON
    "CLANG_TIDY_PATH" OFF)
if(ENABLE_LINTING)
    set_target_properties(h3 PROPERTIES C_CLANG_TIDY "${CLANG_TIDY_PATH}")
elseif(NOT CLANG_TIDY_PATH)
    message(WARNING "clang-tidy was not detected, "
                  "so source code linting is disabled")
endif()

# Docs
find_package(Doxygen)
option(ENABLE_DOCS "Enable building documentation." ON)
if(DOXYGEN_FOUND AND ENABLE_DOCS)
    set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES "${PROJECT_BINARY_DIR}/dev-docs/_build")
    configure_file(dev-docs/Doxyfile.in
        dev-docs/Doxyfile
        ESCAPE_QUOTES
        )
    add_custom_target(docs
        ALL
        ${DOXYGEN_EXECUTABLE} ${PROJECT_BINARY_DIR}/dev-docs/Doxyfile
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/dev-docs
        COMMENT "Generating API documentation with Doxygen" VERBATIM
        )
else()
    add_custom_target(docs
        echo "Doxygen was not installed when CMake was run or ENABLE_DOCS was OFF. Check that Doxygen is installed and rerun `cmake .`" VERBATIM
        )
endif()

# Metadata for bindings
if (WIN32)
    add_custom_target(binding-functions
        COMMAND PowerShell -ExecutionPolicy Bypass -File ${PROJECT_SOURCE_DIR}/scripts/binding_functions.ps1
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
        )
else()
    add_custom_target(binding-functions
        COMMAND ${PROJECT_SOURCE_DIR}/scripts/binding_functions.sh
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
        )
endif()

# Link all executables against H3
macro(add_h3_executable name)
    # invoke built-in add_executable
    add_executable(${ARGV})
    if(TARGET ${name})
        target_link_libraries(${name} PUBLIC h3)
        target_include_directories(${name} PUBLIC
          $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/apps/applib/include>)
    endif()
endmacro()

add_h3_executable(geoToH3 src/apps/filters/geoToH3.c ${APP_SOURCE_FILES})
add_h3_executable(h3ToComponents src/apps/filters/h3ToComponents.c ${APP_SOURCE_FILES})
add_h3_executable(h3ToGeo src/apps/filters/h3ToGeo.c ${APP_SOURCE_FILES})
add_h3_executable(h3ToGeoBoundary src/apps/filters/h3ToGeoBoundary.c ${APP_SOURCE_FILES})
add_h3_executable(hexRange src/apps/filters/hexRange.c ${APP_SOURCE_FILES})
add_h3_executable(kRing src/apps/filters/kRing.c ${APP_SOURCE_FILES})
add_h3_executable(generateBaseCellNeighbors src/apps/miscapps/generateBaseCellNeighbors.c ${APP_SOURCE_FILES})
add_h3_executable(h3ToGeoBoundaryHier src/apps/miscapps/h3ToGeoBoundaryHier.c ${APP_SOURCE_FILES})
add_h3_executable(h3ToGeoHier src/apps/miscapps/h3ToGeoHier.c ${APP_SOURCE_FILES})
add_h3_executable(h3ToHier src/apps/miscapps/h3ToHier.c ${APP_SOURCE_FILES})

# Generate KML files for visualizing the H3 grid
add_custom_target(create-kml-dir
    COMMAND ${CMAKE_COMMAND} -E make_directory KML)
add_custom_target(kml)

# Only the first 3 resolution grids are generated. The others can be generated,
# but the file sizes would be very, very large.
foreach(resolution RANGE 3)
    set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES "KML/res${resolution}cells.kml")
    set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES "KML/res${resolution}centers.kml")
    add_custom_target(kml_cells_${resolution}
        COMMAND ${SHELL} "$<TARGET_FILE:h3ToHier> ${resolution} | $<TARGET_FILE:h3ToGeoBoundary> 1 res${resolution}cells.kml \"Res ${resolution} Cells\" > KML/res${resolution}cells.kml"
        VERBATIM
        DEPENDS create-kml-dir)
    add_custom_target(kml_centers_${resolution}
        COMMAND ${SHELL} "$<TARGET_FILE:h3ToHier> ${resolution} | $<TARGET_FILE:h3ToGeo> 1 res${resolution}centers.kml \"Res ${resolution} Centers\" > KML/res${resolution}centers.kml"
        VERBATIM
        DEPENDS create-kml-dir)
    add_dependencies(kml
        kml_cells_${resolution}
        kml_centers_${resolution})
endforeach()

if(BUILD_TESTING)
    option(PRINT_TEST_FILES "Print which test files correspond to which tests" OFF)

    find_program(VALGRIND valgrind)
    cmake_dependent_option(WRAP_VALGRIND "Wrap tests in valgrind" OFF
                           "VALGRIND" OFF)
    if(WRAP_VALGRIND)
        set(TEST_WRAPPER ${VALGRIND} --track-origins=yes --leak-check=full --error-exitcode=99)
        string(REPLACE ";" " " TEST_WRAPPER_STR "${TEST_WRAPPER}")
    endif()

    enable_testing()

    # Macros and support code needed to build and add the tests
    set(test_number 0)

    option(ENABLE_COVERAGE "Enable compiling tests with coverage." ON)
    if(ENABLE_COVERAGE)
        set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES "coverage.info")
        set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES "coverage.cleaned.info")
        set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES "coverage")
        add_custom_target(coverage
            # Compile generated coverage counts into a single coverage.info
            COMMAND lcov --directory '${PROJECT_BINARY_DIR}' --capture --output-file coverage.info
            # Take only coverage of the core library, not apps
            COMMAND lcov --extract coverage.info '${PROJECT_SOURCE_DIR}/src/h3lib/*' --output-file coverage.cleaned.info
            # Generate output html at coverage/
            COMMAND genhtml -o coverage coverage.cleaned.info --title 'h3 coverage'
            COMMENT "Coverage done"
            )
        add_custom_target(clean-coverage
            # Before running coverage, clear all counters
            COMMAND lcov --directory '${PROJECT_BINARY_DIR}' --zerocounters
            COMMENT "Zeroing counters"
            )
    endif()

    macro(add_h3_test_common name srcfile)
        # need to actually make the test target
        if(NOT TARGET ${name})
            add_h3_executable(${name} ${srcfile} ${APP_SOURCE_FILES})
        endif()

        math(EXPR test_number "${test_number}+1")
    endmacro()

    macro(add_h3_test name srcfile)
        add_h3_test_common(${name} ${srcfile})
        add_test(NAME ${name}_test${test_number} COMMAND ${TEST_WRAPPER} "$<TARGET_FILE:${name}>")

        if(ENABLE_COVERAGE)
            add_custom_target(${name}_coverage${test_number}
                COMMAND ${name} > /dev/null
                COMMENT "Running ${name}_coverage${test_number}"
                )

            add_dependencies(coverage ${name}_coverage${test_number})
            add_dependencies(${name}_coverage${test_number} clean-coverage)
        endif()
    endmacro()

    macro(add_h3_test_with_file name srcfile argfile)
        add_h3_test_common(${name} ${srcfile})
        # add a special command (so we don't need to read the test file from the test program)
        if(WIN32)
            set(dump_command "Get-Content")
        else()
            set(dump_command "cat")
        endif()

        add_test(NAME ${name}_test${test_number}
                 COMMAND ${SHELL} "${dump_command} ${argfile} | ${TEST_WRAPPER_STR} $<TARGET_FILE:${name}>")

        if(PRINT_TEST_FILES)
            message("${name}_test${test_number} - ${argfile}")
        endif()

        if(ENABLE_COVERAGE)
            add_custom_target(${name}_coverage${test_number}
                COMMAND ${name} < ${argfile} > /dev/null
                COMMENT "Running ${name}_coverage${test_number}"
                )

            add_dependencies(coverage ${name}_coverage${test_number})
            add_dependencies(${name}_coverage${test_number} clean-coverage)
        endif()
    endmacro()

    macro(add_h3_test_with_arg name srcfile arg)
        add_h3_test_common(${name} ${srcfile})
        add_test(NAME ${name}_test${test_number}
            COMMAND ${TEST_WRAPPER} $<TARGET_FILE:${name}> ${arg}
            )
        if(PRINT_TEST_FILES)
            message("${name}_test${test_number} - ${arg}")
        endif()

        if(ENABLE_COVERAGE)
            add_custom_target(${name}_coverage${test_number}
                COMMAND ${name} ${arg}
                COMMENT "Running ${name}_coverage${test_number}"
                )

            add_dependencies(coverage ${name}_coverage${test_number})
            add_dependencies(${name}_coverage${test_number} clean-coverage)
        endif()
    endmacro()

    # Add each individual test

    file(GLOB all_centers tests/inputfiles/bc*centers.txt)
    foreach(file ${all_centers})
        add_h3_test_with_file(testH3ToGeo src/apps/testapps/testH3ToGeo.c ${file})
    endforeach()

    file(GLOB all_ic_files tests/inputfiles/res*ic.txt)
    foreach(file ${all_ic_files})
        add_h3_test_with_file(testH3ToGeo src/apps/testapps/testH3ToGeo.c ${file})
    endforeach()

    file(GLOB all_centers tests/inputfiles/rand*centers.txt)
    foreach(file ${all_centers})
        add_h3_test_with_file(testGeoToH3 src/apps/testapps/testGeoToH3.c ${file})
    endforeach()

    file(GLOB all_cells tests/inputfiles/*cells.txt)
    foreach(file ${all_cells})
        add_h3_test_with_file(testH3ToGeoBoundary src/apps/testapps/testH3ToGeoBoundary.c ${file})
    endforeach()

    add_h3_test(testCompact src/apps/testapps/testCompact.c)
    add_h3_test(testKRing src/apps/testapps/testKRing.c)
    add_h3_test(testHexRing src/apps/testapps/testHexRing.c)
    add_h3_test(testHexRanges src/apps/testapps/testHexRanges.c)
    add_h3_test(testH3ToParent src/apps/testapps/testH3ToParent.c)
    add_h3_test(testH3ToChildren src/apps/testapps/testH3ToChildren.c)
    add_h3_test(testMaxH3ToChildrenSize src/apps/testapps/testMaxH3ToChildrenSize.c)
    add_h3_test(testH3Index src/apps/testapps/testH3Index.c)
    add_h3_test(testH3Api src/apps/testapps/testH3Api.c)
    add_h3_test(testH3SetToLinkedGeo src/apps/testapps/testH3SetToLinkedGeo.c)
    add_h3_test(testH3SetToVertexGraph src/apps/testapps/testH3SetToVertexGraph.c)
    add_h3_test(testLinkedGeo src/apps/testapps/testLinkedGeo.c)
    add_h3_test(testPolyfill src/apps/testapps/testPolyfill.c)
    add_h3_test(testVertexGraph src/apps/testapps/testVertexGraph.c)
    add_h3_test(testH3UniEdge src/apps/testapps/testH3UniEdge.c)
    add_h3_test(testGeoCoord src/apps/testapps/testGeoCoord.c)
    add_h3_test(testBBox src/apps/testapps/testBBox.c)

    add_h3_test_with_arg(testH3NeighborRotations src/apps/testapps/testH3NeighborRotations.c 0)
    add_h3_test_with_arg(testH3NeighborRotations src/apps/testapps/testH3NeighborRotations.c 1)
    add_h3_test_with_arg(testH3NeighborRotations src/apps/testapps/testH3NeighborRotations.c 2)

    # Miscellaneous testing applications
    add_h3_executable(mkRandGeo src/apps/testapps/mkRandGeo.c ${APP_SOURCE_FILES})
    add_h3_executable(mkRandGeoBoundary src/apps/testapps/mkRandGeoBoundary.c ${APP_SOURCE_FILES})

    # Benchmarks
    add_custom_target(benchmarks)

    macro(add_h3_benchmark name srcfile)
        add_h3_executable(${name} ${srcfile} ${APP_SOURCE_FILES})
        add_custom_target(bench_${name} COMMAND ${name})
        add_dependencies(benchmarks bench_${name})
    endmacro()

    add_h3_benchmark(benchmarkH3Api src/apps/benchmarks/benchmarkH3Api.c)
    add_h3_benchmark(benchmarkPolyfill src/apps/benchmarks/benchmarkPolyfill.c)
endif()

# Installation (https://github.com/forexample/package-example)

# Layout. This works for all platforms:
#   * <prefix>/lib/cmake/<PROJECT-NAME>
#   * <prefix>/lib/
#   * <prefix>/include/
set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Configuration
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

# Include module with fuction 'write_basic_package_version_file'
include(CMakePackageConfigHelpers)

# Configure '<PROJECT-NAME>ConfigVersion.cmake'
# Use:
#   * PROJECT_VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Configure '<PROJECT-NAME>Config.cmake'
# Use variables:
#   * TARGETS_EXPORT_NAME
#   * PROJECT_NAME
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

# Targets:
#   * <prefix>/lib/libh3.so
#   * header location after install: <prefix>/include/h3/h3api.h
#   * headers can be included by C++ code `#include <h3/h3api.h>`
# Installing the library and filters system-wide.
install(
    TARGETS h3 geoToH3 h3ToComponents h3ToGeo h3ToGeoBoundary hexRange
            kRing h3ToGeoBoundaryHier h3ToGeoHier h3ToHier
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
)

# Headers:
#   * src/h3lib/include/h3api.h -> <prefix>/include/h3/h3api.h
# Only the h3api.h header is needed by applications using H3.
install(
    FILES src/h3lib/include/h3api.h
    DESTINATION "${include_install_dir}/h3"
)

# Config
#   * <prefix>/lib/cmake/h3/h3Config.cmake
#   * <prefix>/lib/cmake/h3/h3ConfigVersion.cmake
install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

# Config
#   * <prefix>/lib/cmake/h3/h3Targets.cmake
install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
