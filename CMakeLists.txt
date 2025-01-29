#
# CMake Boost Build.
#
# Cmd line options:
#   CMAKE_BUILD_TYPE in { Debug, Release }
#   BOOST_VERSION - valid boost version (Example: 1.76.0)
#
#

CMAKE_MINIMUM_REQUIRED(VERSION 3.18)
PROJECT(CMAKE_BOOST CXX)

OPTION(WITHOUT_PYTHON "If ON Boost is build without Python, If off, with python" OFF)

IF(NOT BOOST_VERSION)
    MESSAGE(FATAL_ERROR "Boost version variable BOOST_VERSION not specified!!")
ENDIF()

IF(NOT CMAKE_BUILD_TYPE)
    MESSAGE(FATAL_ERROR "Please set CMAKE_BUILD_TYPE!!")
ENDIF()

IF(NOT "${CMAKE_BUILD_TYPE}" STREQUAL "Debug" AND
        NOT "${CMAKE_BUILD_TYPE}" STREQUAL "Release")
    MESSAGE(FATAL_ERROR "Unsupported build type: ${CMAKE_BUILD_TYPE}")
ENDIF()

STRING(REGEX MATCH "^[0-9]\.[0-9]+\.[0-9]+$" version_ok "${BOOST_VERSION}")
IF(NOT version_ok)
    MESSAGE(FATAL_ERROR "Boost version ${BOOST_VERSION} is not valid!")
ENDIF()

STRING(REPLACE "\." "_" BOOST_VERSION_UNDERSCORE "${BOOST_VERSION}")

SET(BOOST_DOWNLOAD_URL "https://archives.boost.io/release/${BOOST_VERSION}/source/boost_${BOOST_VERSION_UNDERSCORE}.tar.gz")
SET(BOOST_INSTALL_DIR  "${CMAKE_CURRENT_BINARY_DIR}/boost_install")

SET(BOOST_BUILD_OPTIONS "cxxflags=-fPIC" "cflags=-fPIC")
IF(CMAKE_CROSSCOMPILING)
    MESSAGE(STATUS "Cross-compiling mode detected")
    MESSAGE(STATUS "Procesor " ${CMAKE_SYSTEM_PROCESSOR})
    IF (CMAKE_SYSTEM_NAME STREQUAL "Linux")
        SET(BOOST_BUILD_OPTIONS "${BOOST_BUILD_OPTIONS}" "target-os=linux")
    ELSE ()
        MESSAGE(FATAL_ERROR "Cross-compiling is not supported on ${CMAKE_SYSTEM_NAME}")
    ENDIF()

    IF (CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "arm")
        SET(BOOST_BUILD_OPTIONS "${BOOST_BUILD_OPTIONS}" "toolset=gcc-arm")
    ELSE ()
        MESSAGE(FATAL_ERROR "Cross-compiling is not supported on processor ${CMAKE_SYSTEM_PROCESSOR}")
    ENDIF()
ENDIF ()


SET(archive_path           ${CMAKE_CURRENT_BINARY_DIR}/boost_${BOOST_VERSION_UNDERSCORE}.tar.gz)
SET(extracted_archive_path ${CMAKE_CURRENT_BINARY_DIR}/boost_${BOOST_VERSION_UNDERSCORE}_${CMAKE_BUILD_TYPE})

IF(NOT EXISTS "${archive_path}")
    MESSAGE(STATUS "Download URL: ${BOOST_DOWNLOAD_URL}")
    FILE(DOWNLOAD "${BOOST_DOWNLOAD_URL}" "${archive_path}" SHOW_PROGRESS)
ENDIF()
FILE(MAKE_DIRECTORY  "${extracted_archive_path}")
EXECUTE_PROCESS(
    COMMAND ${CMAKE_COMMAND} -E tar xjf ${archive_path}
    WORKING_DIRECTORY ${extracted_archive_path} 
)
SET(boost_sources "${extracted_archive_path}/boost_${BOOST_VERSION_UNDERSCORE}")

MESSAGE(STATUS ${boost_sources})

IF (CMAKE_CROSSCOMPILING)
    # Create file that defines CXX compiler for Boost build
    set(USER_CONFIG_JAM_PATH "${boost_sources}/tools/build/src/user-config.jam")
    file(WRITE ${USER_CONFIG_JAM_PATH} "using gcc : arm : $ENV{CXX} ;\n")
ENDIF()

    SET(extra_b2_args)
IF(WITHOUT_PYTHON)
    LIST(APPEND extra_b2_args "--without-python")
ENDIF()

ADD_CUSTOM_COMMAND(OUTPUT "${boost_sources}/b2"
    MAIN_DEPENDENCY "${boost_sources}"
    COMMAND ./bootstrap.sh
    WORKING_DIRECTORY "${boost_sources}"
    COMMENT "Boost bootstrap"
)

ADD_CUSTOM_COMMAND(
    OUTPUT "${BOOST_INSTALL_DIR}"
    MAIN_DEPENDENCY "${boost_sources}"
    DEPENDS "${boost_sources}/b2"
    COMMAND ./b2
        ${extra_b2_args}
        --prefix="${BOOST_INSTALL_DIR}"
        runtime-link=shared,static
        link=shared,static
        variant=$<$<CONFIG:Debug>:debug>$<$<CONFIG:Release>:release>
        threading=multi 
        ${BOOST_BUILD_OPTIONS}
        address-model=64 --layout=tagged
        -j 16 install
    WORKING_DIRECTORY "${boost_sources}"
    COMMENT "Build Boost from sources"
)

ADD_CUSTOM_TARGET(boost_build ALL
    DEPENDS "${BOOST_INSTALL_DIR}"
)

INSTALL(DIRECTORY ${BOOST_INSTALL_DIR}/include/
    DESTINATION "include"
)

INSTALL(DIRECTORY ${BOOST_INSTALL_DIR}/lib/
    DESTINATION "lib"
)