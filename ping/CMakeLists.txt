cmake_minimum_required(VERSION 3.10.2)
project(ping)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
add_executable(ping main.cpp)