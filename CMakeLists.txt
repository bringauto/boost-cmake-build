#
# CMake Boost Build.
#
# Cmd line options:
#   CMAKE_BUILD_TYPE in { Debug, Release }
#   BOOST_VERSION - valid boost version (Example: 1.76.0)
#
#

PROJECT(CMAKE_BOOST CXX)
CMAKE_MINIMUM_REQUIRED(VERSION 3.16)

IF(NOT BOOST_VERSION)
    MESSAGE(FATAL_ERROR "Boost version veraible BOOST_VERSION not specified!!")
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

SET(BOOST_DOWNLOAD_URL "https://boostorg.jfrog.io/artifactory/main/release/${BOOST_VERSION}/source/boost_${BOOST_VERSION_UNDERSCORE}.tar.gz")
SET(BOOST_INSTALL_DIR "${CMAKE_CURRENT_BINARY_DIR}/boost_install")

SET(BOOST_BUILD_OPTIONS "cxxflags=-fPIC" "cflags=-fPIC")



SET(archive_path           ${CMAKE_CURRENT_BINARY_DIR}/boost_${BOOST_VERSION_UNDERSCORE}.tar.gz)
SET(extracted_archive_path ${CMAKE_CURRENT_BINARY_DIR}/boost_${BOOST_VERSION_UNDERSCORE}_${CMAKE_BUILD_TYPE})

IF(NOT EXISTS "${archive_path}")
    FILE(DOWNLOAD "${BOOST_DOWNLOAD_URL}" "${archive_path}" SHOW_PROGRESS)
ENDIF()
FILE(MAKE_DIRECTORY  "${extracted_archive_path}")
FILE(ARCHIVE_EXTRACT INPUT "${archive_path}" DESTINATION "${extracted_archive_path}")
SET(boost_sources    "${extracted_archive_path}/boost_${BOOST_VERSION_UNDERSCORE}")

MESSAGE(STATUS ${boost_sources})

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
    COMMAND ./b2 --prefix="${BOOST_INSTALL_DIR}"
        runtime-link=shared,static
        link=shared,static
        variant=$<$<CONFIG:Debug>:debug>$<$<CONFIG:Release>:release>
        threading=multi 
        ${BOOST_BUILD_OPTIONS}
        address-model=64 --layout=tagged
        --without-python
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