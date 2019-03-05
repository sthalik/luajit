cmake_minimum_required(VERSION 3.0)

project(minilua)

find_library(LIBM NAMES m)

add_executable(minilua ../../minilua.c)
target_link_libraries(minilua ${LIBM})
