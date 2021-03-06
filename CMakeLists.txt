cmake_minimum_required(VERSION 3.6)
project(fost-boost)


set(BOOST_SEARCH YES
    CACHE STRING "Turn on to stop cmake from searching for a version of Boost")
set(BOOST_DIRECTORY_SUFFIX ""
    CACHE STRING "String appended (with a dash) to the end of Boost's directory")
set(BOOST_TOOLSET ""
    CACHE STRING "Toolset to pass to Boost build")


if(BOOST_SEARCH)
    find_package(Boost 1.62.0)
endif()

if(APPLE)
    set(so-suffix -mt)
    set(so-ext dylib)
else()
    set(so-ext so)
endif()

if(Boost_FOUND AND NOT BOOST_VMINOR)
    add_library(boost INTERFACE)
    target_include_directories(boost INTERFACE ${Boost_INCLUDE_DIRS})
    function(boostlib name)
        if(NOT TARGET boost_${name})
            message(STATUS "Searching system for boost_${name}")
            find_package(Boost 1.62.0 COMPONENTS ${name})
            add_library(boost_${name} SHARED IMPORTED GLOBAL)
            set(so ${Boost_LIBRARY_DIRS}/libboost_${name}${so-suffix}.${so-ext})
            set_property(TARGET boost_${name}
                PROPERTY IMPORTED_LOCATION ${so})
        endif()
    endfunction()
else()
    if(NOT "${BOOST_TOOLSET}" STREQUAL "")
        set(toolset "${BOOST_TOOLSET}")
    elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
        set(toolset "gcc")
    elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" OR ${CMAKE_CXX_COMPILER_ID} STREQUAL "AppleClang")
        set(toolset "clang")
    else()
        message(SEND_ERROR "Unknown compiler ID: ${CMAKE_CXX_COMPILER_ID}")
    endif()

    if("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
        set(build "release")
    else()
        set(build "debug")
    endif()

    set(BOOST_VMAJOR 1
        CACHE STRING "Major part for Boost version. Always 1")
    set(BOOST_VMINOR 66
        CACHE STRING "Minor part for Boost version. Use 62 and up")
    set(BOOST_VPATCH 0
        CACHE STRING "Patch part for Boost version. Normally 0")

    set(boost-ver ${BOOST_VMAJOR}.${BOOST_VMINOR}.${BOOST_VPATCH})
    set(boost-dir boost/${BOOST_VMAJOR}_${BOOST_VMINOR}_${BOOST_VPATCH}/${toolset}/${build}${BOOST_DIRECTORY_SUFFIX})
    set(boost-cmake-dir ${CMAKE_CURRENT_SOURCE_DIR})

    if(NOT EXISTS ${boost-cmake-dir}/${boost-dir}/include)
        execute_process(COMMAND ./download ${BOOST_VMAJOR} ${BOOST_VMINOR} ${BOOST_VPATCH} ${BOOST_DIRECTORY_SUFFIX}
            WORKING_DIRECTORY ${boost-cmake-dir})
        execute_process(COMMAND ./bootstrap ${BOOST_VMAJOR} ${BOOST_VMINOR} ${BOOST_VPATCH} ${BOOST_DIRECTORY_SUFFIX}
            WORKING_DIRECTORY ${boost-cmake-dir})
        execute_process(COMMAND ./compile ${BOOST_VMAJOR} ${BOOST_VMINOR} ${BOOST_VPATCH} ${toolset} ${build} ${BOOST_DIRECTORY_SUFFIX}
            WORKING_DIRECTORY ${boost-cmake-dir})
    else()
        message(STATUS "Locally built Boost already found in " ${boost-dir})
    endif()
    link_directories(${boost-cmake-dir}/${boost-dir}/lib/)

    add_library(boost INTERFACE)
    target_include_directories(boost INTERFACE ${boost-dir}/include/)
    target_compile_definitions(boost INTERFACE BOOST_NO_AUTO_PTR=)

    function(boostlib name)
        if(NOT TARGET boost_${name})
            add_library(boost_${name} SHARED IMPORTED GLOBAL)
            target_link_libraries(boost_${name} INTERFACE boost)
            if(NOT APPLE)
                set(so ${boost-cmake-dir}/${boost-dir}/lib/libboost_${name}.${so-ext}.${boost-ver})
            else()
                set(so ${boost-cmake-dir}/${boost-dir}/lib/libboost_${name}.${so-ext})
            endif()
            set_property(TARGET boost_${name}
                PROPERTY IMPORTED_LOCATION ${so})
            install(FILES ${so} DESTINATION lib)
        endif()
    endfunction()
endif()

boostlib(atomic)
boostlib(chrono)
boostlib(context)
boostlib(coroutine)
boostlib(date_time)
boostlib(filesystem)
boostlib(system)
boostlib(thread)

