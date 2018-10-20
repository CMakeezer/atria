# Copyright (c) 2014, 2015, 2016 Ableton AG, Berlin

cmake_minimum_required(VERSION 3.1.0)

project(Atria C CXX)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
include(FeatureSummary)

include(PrintProperties)
set(Atria_VERSION_MAJOR 0)
set(Atria_VERSION_MINOR 0)
set(CMAKE_VERBOSE_MAKEFILE FALSE)
set(ENABLE_INSTALL TRUE)
option(ENABLE_TEST "" TRUE)
set(PRINT_TARGET_PROPERTIES FALSE)
set(ENABLE_COVERAGE FALSE)
if(ENABLE_COVERAGE)
    include(CodeCoverage)
endif()

find_package(cxxopts REQUIRED)
find_package(eggs.variant REQUIRED)

# Enable C++11
if (NOT CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 11)
endif ()
set(CMAKE_CXX_STANDARD_REQUIRED on)

if (CMAKE_CXX_STANDARD STREQUAL 14)
    add_definitions(-DABL_CXX14=1)
else ()
    add_definitions(-DABL_CXX14=0)
endif ()

# Build debug by default
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to 'Debug' as none was specified.")
    set(CMAKE_BUILD_TYPE Debug)
endif ()

#Enable Unit Test coverage
if (ENABLE_COVERAGE)
    APPEND_COVERAGE_COMPILER_FLAGS()
endif ()

# Enable warnings
if (CMAKE_CXX_COMPILER_ID STREQUAL Clang
        OR CMAKE_CXX_COMPILER_ID STREQUAL AppleClang)
    set(warning_options
            -Werror
            -Weverything
            -Wno-c++98-compat
            -Wno-c++98-compat-pedantic
            -Wno-documentation-unknown-command
            -Wno-exit-time-destructors
            -Wno-global-constructors
            -Wno-padded
            -Wno-string-conversion
            -Wno-undefined-inline
            -Wno-undefined-internal
            -Wno-undefined-reinterpret-cast
            -Wno-unknown-pragmas
            -Wno-weak-vtables
            -Wno-disabled-macro-expansion
            )
elseif (MSVC)
    set(warning_options /WX /W4 /wd4503 /wd4512)
endif ()

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE ON)
set(CMAKE_MACOSX_RPATH ON)

enable_testing()

find_package(Boost 1.54 REQUIRED)
find_package(Threads REQUIRED)
find_package(GTest 1.80 REQUIRED)

add_feature_info(GoogleTest ${GTEST_FOUND} "used for unit testing.")
feature_summary(WHAT ALL FATAL_ON_MISSING_REQUIRED_PACKAGES)

file(GLOB_RECURSE atria_sources "src/*.hpp" "src/*.cpp")
file(GLOB_RECURSE atria_test_sources "src/tst_*.cpp")
file(GLOB_RECURSE atria_main_sources "src/*_main.cpp")
list(REMOVE_ITEM atria_sources ${atria_test_sources})
list(REMOVE_ITEM atria_sources ${atria_main_sources})

# Defining Atria
add_library(atria ${atria_sources})
set_target_properties(atria PROPERTIES LINKER_LANGUAGE CXX)
target_include_directories(atria PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/compat>)
target_compile_options(atria
        PUBLIC
        ${warning_options})
target_link_libraries(atria PRIVATE Boost::boost cxxopts::cxxopts Eggs::Variant)
if (PRINT_TARGET_PROPERTIES)
    print_target_properties(atria)
endif ()

if(BUILD_EXAMPLES)
    add_executable(example-boost-signal examples/xform_boost_signal.cpp)
    target_link_libraries(example-boost-signal atria)
endif()

# Definining tests
if (GTEST_FOUND AND ENABLE_TEST)
    add_executable(atria-test-all
            ${atria_test_sources})
    target_link_libraries(atria-test-all
            atria
            GTest::GTest
            GTest::Main
            ${CMAKE_THREAD_LIBS_INIT})
    add_test(atria-test-all atria-test-all)

    if (ENABLE_COVERAGE)
        SETUP_TARGET_FOR_COVERAGE(
                NAME atria_coverage                    # New target name
                EXECUTABLE atria-test-all -j4#${PROCESSOR_COUNT} # Executable in PROJECT_BINARY_DIR
        )
    endif ()

    add_executable(atria-benchmark-xform
            src/atria/xform/benchmark_main.cpp)
    target_link_libraries(atria-benchmark-xform
            atria)
    add_test(atria-benchmark-xform atria-benchmark-xform)
endif ()


set(include_install_dir "include")
set(lib_install_dir "lib/")
set(bin_install_dir "bin/")
set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(pkgconfig_install_dir "lib/pkgconfig")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(pkg_config "${generated_dir}/${PROJECT_NAME}.pc")
set(targets_export_name "${PROJECT_NAME}Targets")

set(namespace "${PROJECT_NAME}::")

set(PROJECT_VERSION "1.0.0")
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
        "${version_config}" VERSION ${GIT_VERSION} COMPATIBILITY SameMajorVersion
)

configure_file("${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in" "${project_config}" @ONLY)
configure_file("${PROJECT_SOURCE_DIR}/cmake/atria.pc.in" "${pkg_config}" @ONLY)

if (ENABLE_INSTALL)
    # Install target (will install the library to specified CMAKE_INSTALL_PREFIX variable)
    install(
            TARGETS atria
            EXPORT ${targets_export_name}
            ARCHIVE DESTINATION ${lib_install_dir}
            LIBRARY DESTINATION ${lib_install_dir}
            RUNTIME DESTINATION ${bin_install_dir}
            INCLUDES DESTINATION ${include_install_dir})

    install(
            DIRECTORY "${PROJECT_SOURCE_DIR}/src/"
            DESTINATION ${include_install_dir}
            FILES_MATCHING PATTERN "*.*hpp")

install(
            DIRECTORY "${PROJECT_SOURCE_DIR}/compat/"
            DESTINATION ${include_install_dir}
            FILES_MATCHING PATTERN "*.*hpp")


    install(
            FILES "${project_config}" "${version_config}"
            DESTINATION "${config_install_dir}")

    install(
            FILES "${pkg_config}"
            DESTINATION "${pkgconfig_install_dir}")

    install(
            EXPORT "${targets_export_name}"
            NAMESPACE "${namespace}"
            DESTINATION "${config_install_dir}")
endif ()
