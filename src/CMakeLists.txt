# ########################################################################
# Copyright 2013 Advanced Micro Devices, Inc.
# 
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
# 
# http://www.apache.org/licenses/LICENSE-2.0
# 
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# ########################################################################

if( WIN32 )
	# We require 2.8.3 for windows because of a bug in cmake that prevented vs2010 from generating
	# executables properly with multiple periods
	cmake_minimum_required( VERSION 2.8.3 )
else( )
	cmake_minimum_required( VERSION 2.6 )
endif( )

# This becomes the name of the solution file
project( clFFT )

# Define a version for the code
set( CLFFT_VERSION_MAJOR 2 )
set( CLFFT_VERSION_MINOR 1 )
set( CLFFT_VERSION_PATCH 0 )
set( CLFFT_VERSION "${CLFFT_VERSION_MAJOR}.${CLFFT_VERSION_MINOR}.${CLFFT_VERSION_PATCH}")
	
# uncomment these to debug nmake and borland makefiles
#SET(CMAKE_START_TEMP_FILE "")
#SET(CMAKE_END_TEMP_FILE "")
#SET(CMAKE_VERBOSE_MAKEFILE 1)

set( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR} )

# On windows, it's convenient to change the default install prefix such that it does NOT point to 'program files'
# Need to check out CMAKE_RUNTIME_OUTPUT_DIRECTORY variable, and see if that eliminates the need to modify install path
if( CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT )
	set( CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}/package" CACHE PATH "Install path prefix, prepended onto install directories" FORCE )
endif( )

# Set the default of CMAKE_BUILD_TYPE to be release, unless user specifies with -D.  MSVC_IDE does not use CMAKE_BUILD_TYPE
if( NOT MSVC_IDE AND NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE )
endif()

# Options that the user or driver program can set to control various components of the build
option( BUILD_RUNTIME "Build the FFT runtime library" ON )
option( BUILD_CLIENT "Build a command line clFFT client program with a variety of configurable parameters (dependency on Boost)" ON )
option( BUILD_TEST "Build the library testing suite (dependency on google test, Boost, and FFTW)" ON )
option( BUILD_LOADLIBRARIES "Build the optional dynamic load libraries that the FFT runtime will search for" ON )

# If BOOST_ROOT is defined as an environment value, use that value and cache it so it's visible in the cmake-gui.  
# Otherwise, create a sensible default that the user can change
if( DEFINED ENV{BOOST_ROOT} )
	set( BOOST_ROOT $ENV{BOOST_ROOT} CACHE PATH "Environment variable defining the root of the Boost installation" )
else( )
	if( UNIX )
		set( BOOST_ROOT "/usr" CACHE PATH "Modify this variable to point to the root of the Boost installation" )
	else( )
		set( BOOST_ROOT "/Path/To/boost_x_xx_x" CACHE PATH "Modify this variable to point to the root of the Boost installation" )
	endif()
endif( )

# Currently, linux has a problem outputing both narrow and wide characters,
# which happens in our client because openCL only supports narrow characters
if( WIN32 )
	option( UNICODE "Build with Unicode Support" ON )
	if( UNICODE )
		message( STATUS "UNICODE build" )
	endif( )
else()
	set( UNICODE OFF )
	message( STATUS "UNICODE feature disabled on linux" )
endif()

if( MSVC_IDE )
	set( BUILD64 ${CMAKE_CL_64} )
    set_property( GLOBAL PROPERTY USE_FOLDERS TRUE )
else()
	option( BUILD64 "Build a 64-bit product" ON )

	if( IS_DIRECTORY ${PROJECT_SOURCE_DIR}/tests )
		option( CODE_COVERAGE "Build makefiles with code coverage instrumentation" OFF )
		if( CODE_COVERAGE )
			message( STATUS "Code coverage instrumentation on" )
		endif()
	endif()
endif()

# Modify the global find property to help us find libraries like Boost in the correct paths for 64-bit
# Essentially, find_library calls will look for /lib64 instead of /lib; works for windows and linux
if( BUILD64 )
	set_property( GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS TRUE )
	message( STATUS "64bit build - FIND_LIBRARY_USE_LIB64_PATHS TRUE" )
else()
	set_property( GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS FALSE )
	message( STATUS "32bit build - FIND_LIBRARY_USE_LIB64_PATHS FALSE" )
endif()

# Client is built only if boost is found; on windows, we need vs10 or higher
# Find Boost on the system, and configure the type of boost build we want
set( Boost_USE_MULTITHREADED ON )
set( Boost_USE_STATIC_LIBS   ON )
set( Boost_DETAILED_FAILURE_MSG   ON )
set( Boost_DEBUG ON )
set( Boost_ADDITIONAL_VERSIONS "1.46.1" "1.46" "1.44.0" "1.44" )

# On linux, the boost installed in the system always appears to override any user boost installs
if( UNIX )
	set( Boost_NO_SYSTEM_PATHS TRUE )
endif( )

# This will define Boost_FOUND
find_package( Boost 1.33.0 COMPONENTS program_options )
message( STATUS "Boost_PROGRAM_OPTIONS_LIBRARY: ${Boost_PROGRAM_OPTIONS_LIBRARY}" )

# This will define OPENCL_FOUND
find_package( OpenCL )

# This will define FFTW_FOUND
find_package( FFTW )

if( (${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} VERSION_LESS 2.8) ) 
	message( STATUS "Cmake version 2.8 or greater needed to use GTest" )
else()
	# This will define GTEST_FOUND
	find_package( GTest )

	# Hack to get googletest v1.6 to work with vs2012
	if( MSVC11 )
		add_definitions( "/D_VARIADIC_MAX=10" )
	endif( )
endif()

# Enable building of the clACML client if both requested and all dependencies are found
if( BUILD_CLIENT AND Boost_FOUND )
	set( FFT_CLIENT ON )
else( )
	set( FFT_CLIENT OFF )
endif( )

# Enable building of the googletest unit test framework if requested and all dependencies are found
if( BUILD_TEST AND GTEST_FOUND AND Boost_FOUND AND FFTW_FOUND )
	set( UNIT_TEST ON )
else( )
	message( "GoogleTest unit testing will NOT be built" )
	set( UNIT_TEST OFF )
endif( )

# FFLAGS depend on the compiler, grab the compiler name from the path
get_filename_component( C_COMPILER_NAME ${CMAKE_C_COMPILER} NAME_WE )
# message( "C_COMPILER_NAME: " ${C_COMPILER_NAME} )
# message( "CMAKE_C_COMPILER: " ${CMAKE_C_COMPILER} )

# Set common compile and link options
if( C_COMPILER_NAME STREQUAL "cl" )
	# Following options for nMake
	message( STATUS "Detected MSVS Ver: " ${MSVC_VERSION} )
	if( NOT MSVC_IDE )
		message( STATUS "Using an nMake environment to build" )

		# I can't get nmake to work because of faulty /machine:, not sure that this isn't a cmake bug
		# if( BUILD64 )
			# set( CMAKE_EXE_LINKER_FLAGS "/machine:amd64 ${CMAKE_EXE_LINKER_FLAGS}" )
			# set( CMAKE_SHARED_LINKER_FLAGS "/machine:amd64 ${CMAKE_SHARED_LINKER_FLAGS}" )
			# set( CMAKE_MODULE_LINKER_FLAGS  "/machine:amd64 ${CMAKE_MODULE_LINKER_FLAGS }" )
		# else( )
			# set( CMAKE_EXE_LINKER_FLAGS "/machine:i386 ${CMAKE_EXE_LINKER_FLAGS}" )
		# endif( )

	endif( )

elseif( C_COMPILER_NAME STREQUAL "gcc" )
	message( STATUS "Detected GNU fortran compiler." )
	EXEC_PROGRAM( ${CMAKE_CXX_COMPILER} ARGS --version OUTPUT_VARIABLE vnum )
	STRING(REGEX REPLACE ".*([0-9])\\.([0-9])\\.([0-9]).*" "\\1\\2\\3" vnum ${vnum})
	if( ${vnum} STREQUAL "452" )
		# we only want c++0x if we're using gcc 4.5.2
		set( CMAKE_CXX_FLAGS "-std=c++0x ${CMAKE_CXX_FLAGS}" )
	endif()
	
	if( BUILD64 )
		set( CMAKE_CXX_FLAGS "-m64 ${CMAKE_CXX_FLAGS}" )
		set( CMAKE_C_FLAGS "-m64 ${CMAKE_C_FLAGS}" )
	else( )
		set( CMAKE_CXX_FLAGS "-m32 ${CMAKE_CXX_FLAGS}" )
		set( CMAKE_C_FLAGS "-m32 ${CMAKE_C_FLAGS}" )
	endif( )

    if( CODE_COVERAGE )
        set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")
        set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage")
    endif()
else( )
	message( FATAL_ERROR "Compiler name not detected" )
endif( )

# If UNICODE is defined, pass extra definitions into 
if( UNICODE )
	add_definitions( "/DUNICODE /D_UNICODE" )
endif( )

# Print out compiler flags for viewing/debug
message( STATUS "CMAKE_CXX_COMPILER flags: " ${CMAKE_CXX_FLAGS} )
message( STATUS "CMAKE_CXX_COMPILER debug flags: " ${CMAKE_CXX_FLAGS_DEBUG} )
message( STATUS "CMAKE_CXX_COMPILER release flags: " ${CMAKE_CXX_FLAGS_RELEASE} )
message( STATUS "CMAKE_CXX_COMPILER relwithdebinfo flags: " ${CMAKE_CXX_FLAGS_RELWITHDEBINFO} )
message( STATUS "CMAKE_EXE_LINKER link flags: " ${CMAKE_EXE_LINKER_FLAGS} )

# configure a header file to pass the CMake version settings to the source, and package the header files in the output archive
configure_file( "${PROJECT_SOURCE_DIR}/include/version.h.in" "${PROJECT_BINARY_DIR}/include/version.h" )
install( FILES 
			"${PROJECT_BINARY_DIR}/include/version.h" 
			"include/clFFT.h"
			"include/clAmdFft.h"
			"include/clAmdFft.version.h" 
		DESTINATION 
			"./include" )


# Recurse into subdirectory and start building files there
if( BUILD_RUNTIME AND IS_DIRECTORY "${PROJECT_SOURCE_DIR}/library" )
	add_subdirectory( library )
else()
	message( "Runtime library will NOT be built" )
endif( )

if( IS_DIRECTORY "${PROJECT_SOURCE_DIR}/scripts/perf" )
	add_subdirectory( scripts/perf )
endif( )

# We only want to build the following if the user options are set
if( FFT_CLIENT AND IS_DIRECTORY "${PROJECT_SOURCE_DIR}/client" )
	add_subdirectory( client )
else( )
	message( "FFT clients will NOT be built" )
endif( )

# Recurse into subdirectory and start building files there
if( BUILD_LOADLIBRARIES AND IS_DIRECTORY "${PROJECT_SOURCE_DIR}/statTimer" )
	add_subdirectory( statTimer )
else()
	message( "LoadLibraries will NOT be built" )
endif( )

if( UNIT_TEST AND IS_DIRECTORY "${PROJECT_SOURCE_DIR}/tests" )
	# enable_testing( )
	add_subdirectory( tests )
else( )
	message( "GoogleTest unit tests will NOT be built" )
endif( )

# The following code is setting variables to control the behavior of CPack to generate our 
if( WIN32 )
	set( CPACK_SOURCE_GENERATOR "ZIP" )
	set( CPACK_GENERATOR "ZIP" )
else( )
	set( CPACK_SOURCE_GENERATOR "TGZ" )
	set( CPACK_GENERATOR "TGZ" )
endif( )

if( BUILD64 )
	set( CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${CLFFT_VERSION}-${CMAKE_HOST_SYSTEM_NAME}-x64")
else( )
	set( CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${CLFFT_VERSION}-${CMAKE_HOST_SYSTEM_NAME}-x32")
endif( )

set( CPACK_SOURCE_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${CLFFT_VERSION}-${CMAKE_HOST_SYSTEM_NAME}-Source")

set( CPACK_PACKAGE_VERSION_MAJOR ${CLFFT_VERSION_MAJOR} )
set( CPACK_PACKAGE_VERSION_MINOR ${CLFFT_VERSION_MINOR} )
set( CPACK_PACKAGE_VERSION_PATCH ${CLFFT_VERSION_PATCH} )
set( CPACK_PACKAGE_DESCRIPTION_SUMMARY "OpenCL implementation of an FFT library")
set( CPACK_PACKAGE_VENDOR "Neutral")
set( CPACK_SOURCE_IGNORE_FILES "/\\\\.hg/;/\\\\.svn/;/\\\\.git/" )

# Define all variables that influence CPack before including CPack, such as install targets
include( CPack )