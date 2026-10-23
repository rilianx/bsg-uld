cmake_minimum_required(VERSION 3.5)
project(BSG_ULD CXX)

set(CMAKE_CXX_STANDARD 11)
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()
set(CMAKE_CXX_FLAGS_RELEASE "-O3")

add_executable(BSG_ULD
  src/main_uld.cpp
  src/SearchStrategy.cpp
  src/State.cpp
  src/BSG.cpp
  src/Greedy.cpp
  src/Block.cpp
  src/Container.cpp
  src/clpState.cpp
  src/VCS_Function.cpp
  src/ULD.cpp
)

enable_testing()
add_test(NAME uld50 COMMAND BSG_ULD ${CMAKE_CURRENT_SOURCE_DIR}/instances/Final_instances/n50sample0.csv -t 0.5)
