#
# Copyright 2013-2014 Felix Schmitt, Axel Huebl, Rene Widera
#
# This file is part of splash2txt. 
# 
# splash2txt is free software: you can redistribute it and/or modify 
# it under the terms of of either the GNU General Public License or 
# the GNU Lesser General Public License as published by 
# the Free Software Foundation, either version 3 of the License, or 
# (at your option) any later version. 
# splash2txt is distributed in the hope that it will be useful, 
# but WITHOUT ANY WARRANTY; without even the implied warranty of 
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the 
# GNU General Public License and the GNU Lesser General Public License 
# for more details. 
# 
# You should have received a copy of the GNU General Public License 
# and the GNU Lesser General Public License along with splash2txt. 
# If not, see <http://www.gnu.org/licenses/>. 
#

################################################################################
# Required cmake version
################################################################################

cmake_minimum_required(VERSION 2.8.5)


################################################################################
# Project 
################################################################################

project(splash2txt)

set(CMAKE_PREFIX_PATH "/usr/lib/x86_64-linux-gnu/" "$ENV{BOOST_ROOT}" "$ENV{HDF5_ROOT}"
    "$ENV{MPI_ROOT}")

# own modules for find_packages
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}
    ${CMAKE_CURRENT_SOURCE_DIR}/../../cmake/)

include_directories(include)


################################################################################
# Build type (debug, release, relwithdebinfo)
################################################################################

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Debug' as none was specified.")
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo")
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")

if(CMAKE_BUILD_TYPE MATCHES Debug)
elseif(CMAKE_BUILD_TYPE MATCHES Release)
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Werror")
elseif(CMAKE_BUILD_TYPE MATCHES RelWithDebInfo)
  set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} -Werror")
else()
  message(FATAL_ERROR "Unrecognized build type")
endif()

message(STATUS "Building ${CMAKE_BUILD_TYPE}")

################################################################################
# Find MPI
################################################################################

find_package(MPI)
include_directories(SYSTEM ${MPI_C_INCLUDE_PATH})
set(LIBS ${LIBS} ${MPI_C_LIBRARIES})

# bullxmpi fails if it can not find its c++ counter part
if(MPI_CXX_FOUND) 
    set(LIBS ${LIBS} ${MPI_CXX_LIBRARIES})
endif(MPI_CXX_FOUND)


################################################################################
# libSplash (+ hdf5 due to required headers)
################################################################################

# find libSplash installation
# prefer static libraries over shared ones (but do not force them)
set(Splash_USE_STATIC_LIBS ON)
find_package(Splash 1.2.4 REQUIRED COMPONENTS PARALLEL)

if(Splash_FOUND)
    include_directories(SYSTEM ${Splash_INCLUDE_DIRS})
    list(APPEND Splash_DEFINITIONS "-DENABLE_HDF5=1")
    add_definitions(${Splash_DEFINITIONS})
    message(STATUS "libSplash compiler definitions: ${Splash_DEFINITIONS}")
    set(LIBS ${LIBS} ${Splash_LIBRARIES})
endif(Splash_FOUND)


################################################################################
# ADIOS
################################################################################

# find adios installation
#   set(ADIOS_USE_STATIC_LIBS ON) # force static linking
find_package(ADIOS 1.6.0)

if(ADIOS_FOUND)
    add_definitions(-DENABLE_ADIOS=1)

    include_directories(SYSTEM ${ADIOS_INCLUDE_DIRS})
    set(LIBS ${LIBS} ${ADIOS_LIBRARIES})
endif(ADIOS_FOUND)


################################################################################
# Find Boost
################################################################################

find_package(Boost REQUIRED COMPONENTS program_options regex)

include_directories(SYSTEM ${Boost_INCLUDE_DIRS})
link_directories(${Boost_LIBRARY_DIR})
set(LIBS ${LIBS} ${Boost_LIBRARIES})


################################################################################
# Compile & Link splash2txt
################################################################################

set(SRCFILES "splash2txt.cpp")

if(Splash_FOUND)
    list(APPEND SRCFILES "tools_splash_parallel.cpp")
endif(Splash_FOUND)
if(ADIOS_FOUND)
    list(APPEND SRCFILES "tools_adios_parallel.cpp")
endif(ADIOS_FOUND)

add_executable(splash2txt
     ${SRCFILES}
     )

target_link_libraries(splash2txt m ${LIBS})


################################################################################
# Install
################################################################################

install(TARGETS splash2txt RUNTIME DESTINATION .)
