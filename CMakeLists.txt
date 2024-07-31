cmake_minimum_required(VERSION 3.12)
project(onionpir VERSION 0.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)

find_package(SEAL 3.5.1 EXACT REQUIRED PATHS /usr/local/lib/libseal-3.5.a)
include_directories("/usr/local/include")


if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
            STRINGS "Release" "Debug" "MinSizeRel" "RelWithDebInfo")
endif ()
message(STATUS "Build type (CMAKE_BUILD_TYPE): ${CMAKE_BUILD_TYPE}")
message(STATUS "Build type (CMAKE_BUILD_TYPE): ${CMAKE_BUILD_TYPE}")

option(NFL_OPTIMIZED "Build an optimized version of NFL" OFF)
if (NFL_OPTIMIZED)
    message(STATUS "to use optimized version of NFL")

    add_definitions(-DNFL_OPTIMIZED)

    include(CheckCXXSourceCompiles)
    set(CMAKE_CXX_FLAGS_OLD ${CMAKE_CXX_FLAGS})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")

    check_cxx_source_compiles("#include <immintrin.h>\nint main() { return 0;}" NFLLIB_USE_AVX)
    check_cxx_source_compiles("#include <pmmintrin.h>\nint main() { return 0;}" NFLLIB_USE_SSE)
    if (NFLLIB_USE_AVX)
        message(STATUS "Using AVX vector engine")
        add_definitions(-DNTT_AVX2)
    elseif (NFLLIB_USE_SSE)
        message(STATUS "Using SSE vector engine")
        add_definitions(-DNTT_SSE)
    else ()
        message(ERROR "Asking for optimized mode but neither SSE not AVX support detected")
    endif ()
endif ()

add_executable(onionpir main.cpp external_prod.h external_prod.cpp waksman.h waksman.cpp util.h pir_server.h pir_server.cpp pir.cpp pir.h pir_client.cpp pir_client.h)

target_link_libraries(onionpir PRIVATE SEAL::seal)
#for mac
#target_link_libraries(external_prod PRIVATE /usr/local/lib/libnfllib.dylib)

target_link_libraries(onionpir PRIVATE /usr/local/lib/libnfllib_static.a)

# create the library to export
set(OnionPIR_Sources
        main.cpp
        external_prod.cpp
        waksman.cpp
        pir_server.cpp
        pir.cpp
        pir_client.cpp
)

add_library(OnionPIRLib STATIC ${OnionPIR_Sources})

target_include_directories(OnionPIRLib PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include>
)

target_link_libraries(OnionPIRLib PRIVATE SEAL::seal /usr/local/lib/libnfllib_static.a)

set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_SOURCE_DIR}/../../install" CACHE PATH "Installation Directory" FORCE)

install(TARGETS OnionPIRLib
        EXPORT OnionPIRLibTargets
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
        RUNTIME DESTINATION bin)

install(FILES pir.h pir_client.h pir_server.h tools.h util.h waksman.h DESTINATION include)

install(EXPORT OnionPIRLibTargets
        FILE OnionPIRLibConfig.cmake
        DESTINATION lib/cmake/OnionPIRLib)
