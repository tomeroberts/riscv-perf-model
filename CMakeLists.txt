cmake_minimum_required(VERSION 3.19)
set (CMAKE_CXX_STANDARD 17)
set (CMAKE_CXX_STANDARD_REQUIRED ON)

project(olympia CXX)

add_compile_options ( -Werror
  -Wall -Wextra -Winline -Winit-self -Wno-unused-function
  -Wuninitialized -Wno-sequence-point -Wno-inline -Wno-unknown-pragmas
  -Woverloaded-virtual -Wno-unused-parameter -Wno-missing-field-initializers -pipe)

################################################################################
# Set up Sparta
if(IS_DIRECTORY ${SPARTA_SEARCH_DIR})
  message (STATUS "Using '${SPARTA_SEARCH_DIR}' for sparta install")
elseif(DEFINED ENV{CONDA_PREFIX})
  message (STATUS "Looking for SPARTA in the conda environment: '$ENV{CONDA_PREFIX}'")
  set(SPARTA_SEARCH_DIR $ENV{CONDA_PREFIX})
else()
  message (STATUS "If needed, please provide the location where sparta is installed: -DSPARTA_SEARCH_DIR=<directory>")
endif()
set(CMAKE_MODULE_PATH "${SPARTA_SEARCH_DIR}/lib/cmake/sparta" ${CMAKE_MODULE_PATH})

find_package(Sparta REQUIRED)

if (NOT SPARTA_FOUND)
  message (FATAL_ERROR "Sparta was not found in ${SPARTA_SEARCH_DIR}")
else()
  message (STATUS "Sparta was found in ${SPARTA_SEARCH_DIR}")
endif()

include_directories(${SPARTA_INCLUDE_DIRS})

################################################################################
# Set up STF library
set (STF_LIB_BASE ${PROJECT_SOURCE_DIR}/stf_lib)
set (DISABLE_STF_DOXYGEN ON)

if (CMAKE_BUILD_TYPE MATCHES "^[Rr]elease")
  set (FULL_LTO true)
  include(${STF_LIB_BASE}/cmake/stf_linker_setup.cmake)
  setup_stf_linker(false)
endif()



# Use ccache if installed
find_program (CCACHE_PROGRAM ccache)
if (CCACHE_PROGRAM)
  set_property (GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
  message ("-- Using ccache")
endif ()

set (SIM_BASE ${PROJECT_SOURCE_DIR})
set (OLYMPIA_VERSION "v0.1.0")
add_definitions(-DOLYMPIA_VERSION=\"${OLYMPIA_VERSION}\")

if (CMAKE_BUILD_TYPE MATCHES "^[Rr]elease")
  set(SPARTA_BUILD_TYPE "release")
elseif (CMAKE_BUILD_TYPE MATCHES "^[Ff]ast[Dd]ebug")
  set(SPARTA_BUILD_TYPE "release")
elseif (CMAKE_BUILD_TYPE MATCHES "^[Dd]ebug")
  set(SPARTA_BUILD_TYPE "debug")
elseif (CMAKE_BUILD_TYPE MATCHES "^[Pp]rofile")
  set(SPARTA_BUILD_TYPE "release")
else()
  message (FATAL_ERROR "Please provide a CMAKE_BUILD_TYPE: -DCMAKE_BUILD_TYPE=Release|FastDebug|Debug|Profile")
endif()

# Profile build flags
set(CMAKE_CXX_FLAGS_PROFILE   "-O3 -pg -g -ftime-trace")
set(CMAKE_CXX_FLAGS_FASTDEBUG "-O3 -g")
set(CMAKE_CXX_FLAGS_DEBUG     "-O0 -g")

# Include directories
include_directories (core mss)
include_directories (SYSTEM mavis)
include_directories (SYSTEM stf_lib)

# The Core, MSS, and the simulator
add_subdirectory(core)
add_subdirectory(mss)

# Add STF library to the build
add_subdirectory (${STF_LIB_BASE})

# Add testing, but do not build as part of the 'all' target
add_subdirectory (test EXCLUDE_FROM_ALL)

# The simulator
add_executable(olympia
  sim/OlympiaSim.cpp
  sim/main.cpp
  )
target_link_libraries (olympia core mss SPARTA::sparta ${STF_LINK_LIBS})
if (CMAKE_BUILD_TYPE MATCHES "^[Rr]elease")
  target_compile_options (core    PUBLIC -flto)
  target_compile_options (mss     PUBLIC -flto)
  target_compile_options (olympia PUBLIC -flto)
  target_link_options    (olympia PUBLIC -flto)
endif()

# Create a few links like reports and arch directories
file(CREATE_LINK ${PROJECT_SOURCE_DIR}/reports    ${CMAKE_CURRENT_BINARY_DIR}/reports         SYMBOLIC)
file(CREATE_LINK ${PROJECT_SOURCE_DIR}/arches     ${CMAKE_CURRENT_BINARY_DIR}/arches          SYMBOLIC)
file(CREATE_LINK ${PROJECT_SOURCE_DIR}/mavis/json ${CMAKE_CURRENT_BINARY_DIR}/mavis_isa_files SYMBOLIC)
file(CREATE_LINK ${PROJECT_SOURCE_DIR}/traces     ${CMAKE_CURRENT_BINARY_DIR}/traces          SYMBOLIC)
