cmake_minimum_required(VERSION 3.0)

project(minilua)

find_library(LIBM_LIBRARIES NAMES m)

add_executable(minilua ../../minilua.c)
if (LIBM_LIBRARIES)
  target_link_libraries(minilua ${LIBM_LIBRARIES})
endif()

if (MSVC)
  set_target_properties(minilua PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
endif()
