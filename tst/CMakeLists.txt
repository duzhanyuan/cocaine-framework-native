CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/tst/cmake")

# Prepare testing infrastructure.
INCLUDE(DownloadGoogleTesting)

DOWNLOAD_GOOGLE_TESTING()

INCLUDE_DIRECTORIES(
    SYSTEM ${GTEST_INCLUDE_DIR}
    SYSTEM ${GMOCK_INCLUDE_DIR}
)

LINK_DIRECTORIES(${GTEST_BINARY_DIR} ${GMOCK_BINARY_DIR})

INCLUDE_DIRECTORIES(
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/../cocaine-core/cocaine-core/include
    ${CMAKE_SOURCE_DIR}/../cocaine-core/cocaine-core/foreign/asio/asio/include
    ${CMAKE_SOURCE_DIR}/../cocaine-core/cocaine-core/foreign/blackhole/src
)
LINK_DIRECTORIES(${CMAKE_SOURCE_DIR}/../cocaine-core/cocaine-core-build/)

SET(Boost_USE_MULTITHREADED ON)
SET(BOOST_COMPONENTS
    system
    thread
)

FIND_PACKAGE(Boost COMPONENTS ${BOOST_COMPONENTS} REQUIRED)

SET(PROJECT cocaine-framework-cpp-tests)
SET(HEADERS)
SET(SOURCES main service)

PROJECT(${PROJECT})

ADD_EXECUTABLE(${PROJECT} ${SOURCES})
ADD_DEPENDENCIES(${PROJECT} googlemock)

TARGET_LINK_LIBRARIES(${PROJECT}
    ${CMAKE_THREAD_LIBS_INIT}
    ${Boost_LIBRARIES}
    cocaine-core
    gmock
    gtest
    gmock_main
)

ADD_DEFINITIONS(-std=c++11)