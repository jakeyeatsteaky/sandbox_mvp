cmake_minimum_required(VERSION 3.20)
set(PROJECT_NAME sandbox_mvp)
set(PROJECT_TESTS_NAME ${PROJECT_NAME}_tests)

project(${PROJECT_NAME} LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build." FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set(HOST_PLATFORM ${CMAKE_SYSTEM_NAME})
message("Building for platform: ${HOST_PLATFORM}")

set(USE_ADDRESS_SANITIZER OFF CACHE BOOL "" FORCE)
set(USE_HARDMODE ON CACHE BOOL "" FORCE)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

message("Using address sanitizer set to: ${USE_ADDRESS_SANITIZER}")
message("Using compiler flags set to: ${USE_HARDMODE}")
message("Update options in CMake cache")


# ----- Specify Which Challenges to enable -----
file(GLOB_RECURSE SOURCES "src/*.cpp")
file(GLOB_RECURSE HEADERS "include/*.hpp")
file(GLOB_RECURSE TEST_SRCS "tests/*.cpp")

add_subdirectory(external/raylib)

add_executable(${PROJECT_NAME} main.cpp ${SOURCES} ${HEADERS})
target_include_directories(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCE_DIR}/include)
target_link_libraries(${PROJECT_NAME} PRIVATE raylib)

enable_testing()

include(FetchContent)
FetchContent_Declare(
        googletest
        GIT_REPOSITORY https://github.com/google/googletest.git
        GIT_TAG        v1.17.0
)

set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

include(GoogleTest)

function(register_test NAME)
    add_executable(${NAME} ${ARGN})
    target_include_directories(${NAME} PRIVATE ${PROJECT_SOURCE_DIR}/include)
    target_link_libraries(${NAME} PRIVATE GTest::gtest_main)
    gtest_discover_tests(${NAME})
endfunction()

register_test(${PROJECT_TESTS_NAME} ${TEST_SRCS} ${SOURCES} ${HEADERS})

add_custom_target(run
    COMMAND $<TARGET_FILE:${PROJECT_NAME}>
    DEPENDS ${PROJECT_NAME}
    WORKING_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
)

add_custom_target(run_tests
    COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
    DEPENDS ${PROJECT_TESTS_NAME}
)

if (USE_HARDMODE)
    if (MSVC)
        add_compile_options(
            /W4
            /permissive-
            /Zc:__cplusplus
            /EHsc
        )
    else()
        add_compile_options(
            -g
            -Wall
            -Wextra
            -Wpedantic
            -Wconversion
            -Wsign-conversion
            -Wshadow
            -Wnon-virtual-dtor
            -Woverloaded-virtual
            -Wold-style-cast
            -Wnull-dereference
            -Wdouble-promotion
            -Wformat=2
        )
    endif()
endif()

if (USE_ADDRESS_SANITIZER)
    if (CMAKE_BUILD_TYPE STREQUAL "Debug" AND NOT MSVC)
        add_compile_options(
            -g
            -fno-omit-frame-pointer
            -fsanitize=address,undefined
        )
        add_link_options(
            -fsanitize=address,undefined
        )
    endif()
endif()
