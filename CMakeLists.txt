# This code is released under the
# Apache License Version 2.0 http://www.apache.org/licenses/.
#
# Copyright (c) 2012 Louis Dionne
#
cmake_minimum_required(VERSION 3.3.0 FATAL_ERROR)
set (CMAKE_CXX_STANDARD 11) # for constexpr specifier and other goodies

if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "No build type selected, default to Release")
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()
MESSAGE( STATUS "CMAKE_BUILD_TYPE: " ${CMAKE_BUILD_TYPE} )

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules")
include(AppendCompilerFlags)


project(SIMDCompressionAndIntersection VERSION 0.1.1 LANGUAGES CXX C)
set(PROJECT_URL "https://github.com/lemire/SIMDCompressionAndIntersection")
set(PROJECT_DESCRIPTION "A C++ library to compress and intersect sorted lists of integers using SIMD instructions")

include(DetectCPUFeatures)
#
# Runs compiler with "-dumpversion" and parses major/minor
# version with a regex.
#
# Taken&Modified from Boost.cmake
#
function(CXX_COMPILER_DUMPVERSION _OUTPUT_VERSION)

    exec_program(${CMAKE_CXX_COMPILER}
            ARGS ${CMAKE_CXX_COMPILER_ARG1} -dumpversion
            OUTPUT_VARIABLE COMPILER_VERSION
            )
    #string(REGEX REPLACE "([0-9])\\.([0-9])(\\.[0-9])?" "\\1\\2"
    #   COMPILER_VERSION ${COMPILER_VERSION})

    set(${_OUTPUT_VERSION} ${COMPILER_VERSION} PARENT_SCOPE)
endfunction()

if(NOT WIN32)
    CXX_COMPILER_DUMPVERSION(CXX_COMPILER_VERSION)
endif()

MESSAGE( STATUS "CMAKE_SIZEOF_VOID_P (should be 8): " ${CMAKE_SIZEOF_VOID_P} )
if( CMAKE_SIZEOF_VOID_P EQUAL 8 )
    MESSAGE( STATUS "Good. You appear to have a 64-bit system. " )
else()
    MESSAGE( STATUS "Please use a 64-bit system. " )
endif()
MESSAGE( STATUS "CMAKE_CXX_COMPILER_ID: " ${CMAKE_CXX_COMPILER_ID} )
MESSAGE( STATUS "CMAKE_C_COMPILER: " ${CMAKE_C_COMPILER} )
MESSAGE( STATUS "CXX_COMPILER_VERSION: " ${CXX_COMPILER_VERSION} )
if( SUPPORT_SSE42 )
    MESSAGE( STATUS "SSE 4.2 support detected" )
else()
    MESSAGE( STATUS "SSE 4.2 support not detected" )
endif()

if(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
    # require at least gcc 4.7
    if (CXX_COMPILER_VERSION VERSION_LESS 4.7)
        message(STATUS "GCC version must be at least 4.7!")
    endif()
    # Uncomment the following lines to see how the code compiles without AVX,SSE4.2 and/or SSE2
    #set (CMAKE_CXX_FLAGS_RELEASE "-Wall -Ofast -lm  -DNDEBUG -std=c++11 -DHAVE_CXX0X -march=x86-64")
    #set (CMAKE_CXX_FLAGS_RELEASE "-Wall -Ofast -lm  -DNDEBUG -std=c++11 -DHAVE_CXX0X -march=core2")
    #set (CMAKE_CXX_FLAGS_RELEASE "-Wall -Ofast -lm  -DNDEBUG -std=c++11 -DHAVE_CXX0X -msse4.2")
    set (CMAKE_CXX_FLAGS_RELEASE "-fpic -mavx -std=c++11  -Weffc++ -pedantic -O3 -Wall -Wextra")
    set (CMAKE_CXX_FLAGS_DEBUG   "-fpic -mavx -std=c++11  -Weffc++ -pedantic -ggdb -DDEBUG=1 -D_GLIBCXX_DEBUG -Wall -Wextra")

    set (CMAKE_C_FLAGS_RELEASE "-fpic -mavx -std=c99 -pedantic -O3 -Wall -Wextra")
    set (CMAKE_C_FLAGS_DEBUG   "-fpic -mavx -std=c99  -pedantic -ggdb -DDEBUG=1 -D_GLIBCXX_DEBUG -Wall -Wextra")
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "Intel")
    if (CXX_COMPILER_VERSION VERSION_LESS 14.0.1)
        message(STATUS "Intel version must be at least 14.0.1!")
    endif()
    set (CMAKE_CXX_FLAGS_RELEASE "-fpic -std=c++11 -O2 -Wall -ansi -xAVX -DNDEBUG=1  -ggdb")
    set (CMAKE_CXX_FLAGS_DEBUG   "-fpic -std=c++11 -O3 -Wall -ansi -xAVX -DDEBUG=1 -D_GLIBCXX_DEBUG -ggdb")

    set (CMAKE_C_FLAGS_RELEASE "-fpic -std=c99 -O2 -Wall -ansi -xAVX -DNDEBUG=1  -ggdb")
    set (CMAKE_C_FLAGS_DEBUG   "-fpic -std=c99 -O3 -Wall -ansi -xAVX -DDEBUG=1 -D_GLIBCXX_DEBUG   -ggdb")
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" OR ${CMAKE_CXX_COMPILER_ID} STREQUAL "AppleClang")
    if (CXX_COMPILER_VERSION VERSION_LESS 4.2.1)
        message(STATUS  "Clang version must be at least 4.2.1!" )
    endif()
    #set (CMAKE_CXX_FLAGS_RELEASE "-Wall -Wcast-align -O3 -DNDEBUG -std=c++11 -DHAVE_CXX0X -msse4.1 -march=native")
    #set (CMAKE_CXX_FLAGS_DEBUG   "-Wall -Wcast-align -ggdb  -std=c++11 -DHAVE_CXX0X -msse4.1 -march=native")
    #set (CMAKE_C_FLAGS_RELEASE "-Wall -Wcast-align -O3 -DNDEBUG -std=c99 -msse4.1 -march=native")
    #set (CMAKE_C_FLAGS_DEBUG   "-Wall -Wcast-align -ggdb  -std=c99 -msse4.1 -march=native")
elseif(WIN32)
    # TODO add support for later versions?
    if(NOT MSVC12)
        message(STATUS "On Windows, only MSVC version 12 is supported!")
    endif()
else ()
    message(FATAL_ERROR "Please, use GCC, Clang, or the Intel compiler!")
endif()

MESSAGE( STATUS "CMAKE_CXX_FLAGS_DEBUG: " ${CMAKE_CXX_FLAGS_DEBUG} )
MESSAGE( STATUS "CMAKE_CXX_FLAGS_RELEASE: " ${CMAKE_CXX_FLAGS_RELEASE} )
MESSAGE( STATUS "CMAKE_C_FLAGS_DEBUG: " ${CMAKE_C_FLAGS_DEBUG} )
MESSAGE( STATUS "CMAKE_C_FLAGS_RELEASE: " ${CMAKE_C_FLAGS_RELEASE} )

# library target
include_directories(include)
add_library(SIMDCompressionAndIntersection STATIC
        src/bitpacking.cpp
        src/integratedbitpacking.cpp
        src/simdbitpacking.cpp
        src/usimdbitpacking.cpp
        src/simdintegratedbitpacking.cpp
        src/intersection.cpp
        src/varintdecode.c
        src/streamvbyte.c
        src/simdpackedsearch.c
        src/simdpackedselect.c
        src/frameofreference.cpp
        src/for.c
        ${HEADERS}
        )

target_include_directories(SIMDCompressionAndIntersection PUBLIC ${CMAKE_CURRENT_LIST_DIR}/include)