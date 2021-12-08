cmake_minimum_required(VERSION 3.22)

if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(PERF_MACOS_STANDALONE ON)
    project(perf-macos CXX)
else()
    set(PERF_MACOS_STANDALONE OFF)
endif()

add_library(perf-macos INTERFACE include/perf-macos.hpp)
target_include_directories(perf-macos INTERFACE include)

add_executable(perf-macos-test test.cpp)
target_link_libraries(perf-macos-test perf-macos)
