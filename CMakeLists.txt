cmake_minimum_required(VERSION 2.8 FATAL_ERROR)
project(ALPSCoreCTHYB CXX)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_SOURCE_DIR})

# The project relies on the ALPSCore package. If ALPSCore is not found
# automatically, specify its location using:
# export ALPSCore_DIR=/location/to/ALPSCORE/
find_package(ALPSCore REQUIRED)
if (MEASURE_TIMING)
    message("measurement of timing enabled")
    add_definitions(-DMEASURE_TIMING)
    find_package(Boost 1.54.0 REQUIRED timer)
else()
    find_package(Boost 1.54.0)
endif()
if (ALPS_ENABLE_MPI)
    find_package(MPI REQUIRED)
endif()
find_package(Eigen3 REQUIRED)

# Option (use quad precision for part of calculations)
option(USE_QUAD_PRECISION "Use quad precision for part of calculations" OFF)
if(USE_QUAD_PRECISION)
    add_definitions(-DUSE_QUAD_PRECISION)
endif()

#ALPSCore disable debug for gf library
#(please do not set NDEBUG for DEBUG build. That would would disable runtime checks in the solver)
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} -DBOOST_DISABLE_ASSERTS -DNDEBUG")

#include directories
set(CHYB_LIBRARY_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIR} ${MPI_CXX_INCLUDE_PATH}) #rest taken care of by libraries dependencies 
include_directories(${CHYB_LIBRARY_INCLUDE_DIRS})

#source files
set(LIB_FILES ./src/sliding_window.cpp ./src/legendre.cpp ./src/model.cpp ./src/model_detail/clustering.cpp src/operator_util.cpp ./src/util.cpp)
ADD_LIBRARY(hyb ${LIB_FILES})

#Compiler dependent libraries
set(EXTRA_LIBS "")
if (USE_QUAD_PRECISION)
    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
        list(APPEND EXTRA_LIBS "quadmath")
    endif()
    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
        set(CMAKE_CXX_FLAGS " -Qoption,cpp,--extended_float_type ${CMAKE_CXX_FLAGS}")
    endif()
endif()

#Set link libraries
target_link_libraries(hyb ${ALPSCore_LIBRARIES} ${MPI_CXX_LIBRARIES} ${Boost_LIBRARIES} ${EXTRA_LIBS})

#executable
ADD_EXECUTABLE(hybmat ./src/main.cpp)
target_link_libraries(hybmat hyb)

#testing setup
option(Testing "Enable testing" ON)
include(EnableGtests) #defined in ./cmake
include_directories("test")
list(APPEND LINK_ALL hyb)

#testing source files
set(unittest_src
unittest
)

enable_testing(test)

foreach(test ${unittest_src})
    add_gtest(${test} test)
endforeach(test)

#installation
install (TARGETS hybmat DESTINATION bin)
