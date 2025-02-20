# Copyright 2023 Ingemar Hedvall
# SPDX-License-Identifier: MIT
cmake_minimum_required(VERSION 3.20)
project(dbcLib
        VERSION 1.0
        DESCRIPTION "CAN DBC C++ library."
        LANGUAGES CXX)

option(CMAKE_SHARED_LIBS "Static libraries are preferred" OFF)
option(DBC_DOC "If doxygen is installed, then build documentation in Release mode" OFF)
option(DBC_TOOLS "Building applications" ON)
option(DBC_TEST "Building unit test" ON)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_DEBUG_POSTFIX d)

include(CMakePrintHelpers)

if (DBC_TOOLS)
include(script/wxwidgets.cmake)
endif()

if (DBC_TEST)
include(script/googletest.cmake)
endif()

if (DBC_DOC)
include(script/doxygen.cmake)
endif()

include(script/flex.cmake)
include(script/bison.cmake)

if (BISON_FOUND)
    BISON_TARGET(DBC_PARSER src/dbcparser.y ${CMAKE_CURRENT_SOURCE_DIR}/src/dbcparser.cpp
            COMPILE_FLAGS "")
    cmake_print_variables(BISON_DBC_PARSER_DEFINED
            BISON_DBC_PARSER_INPUT
            BISON_DBC_PARSER_OUTPUT_SOURCE
            BISON_DBC_PARSER_OUTPUT_HEADER
            BISON_DBC_PARSER_OUTPUTS
            BISON_DBC_PARSER_COMPILE_FLAGS
            BISON_INCLUDE_DIRS)
endif ()

if (FLEX_FOUND)
    FLEX_TARGET(DBC_SCANNER src/dbcflexer.l ${CMAKE_CURRENT_SOURCE_DIR}/src/dbcflexer.cpp)
    cmake_print_variables(FLEX_DBC_SCANNER_DEFINED
            FLEX_DBC_SCANNER_OUTPUTS
            FLEX_DBC_SCANNER_INPUT
            FLEX_DBC_SCANNER_OUTPUT_HEADER
            FLEX_INCLUDE_DIRS)
endif ()

ADD_FLEX_BISON_DEPENDENCY(DBC_SCANNER DBC_PARSER)
set(DBCL_HEADERS
    include/dbc/attribute.h
    include/dbc/network.h
    include/dbc/envvar.h
    include/dbc/node.h
    include/dbc/message.h
    include/dbc/signal.h
    include/dbc/signalgroup.h
    include/dbc/dbcfile.h
    include/dbc/dbcmessage.h
    include/dbc/isampleobserver.h
    include/dbc/signalobserver.h
)

add_library(dbc STATIC
        ${BISON_DBC_PARSER_OUTPUTS}
        ${FLEX_DBC_SCANNER_OUTPUTS}
        ${DBCL_HEADERS} # not needed for cmake 3.23 and above, because FILE_SET is used
        src/attribute.cpp
        src/network.cpp
        src/envvar.cpp
        src/node.cpp
        src/message.cpp
        src/signal.cpp
        src/signalgroup.cpp
        src/dbcscanner.cpp src/dbcscanner.h
        src/dbcfile.cpp
        src/dbchelper.cpp src/dbchelper.h src/dbcmessage.cpp  src/signalobserver.cpp)

target_include_directories(dbc PUBLIC
        $<INSTALL_INTERFACE:include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        )
target_include_directories(dbc PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
# target_include_directories(dbc PRIVATE ${Boost_INCLUDE_DIRS})
#target_include_directories(dbc PRIVATE ${FLEX_INCLUDE_DIRS})

cmake_print_properties(TARGETS dbc PROPERTIES INCLUDE_DIRECTORIES)

if (MSVC)
    target_compile_definitions(dbc PRIVATE _WIN32_WINNT=0x0A00)
endif ()

if (DBC_TOOLS)
#    add_subdirectory(listend)
endif ()

if (wxWidgets_FOUND AND DBC_TOOLS)
    add_subdirectory(dbcviewer)
endif ()

if (GTEST_FOUND AND DBC_TEST)
    enable_testing()
    add_subdirectory(test)
endif ()

if (DOXYGEN_FOUND AND (CMAKE_BUILD_TYPE MATCHES "^[Rr]elease") AND DBC_DOC)
    set(DOXYGEN_RECURSIVE NO)
    set(DOXYGEN_REPEAT_BRIEF NO)
    set(DOXYGEN_PROJECT_NAME "DbcLib")
    set(DOXYGEN_HTML_EXTRA_STYLESHEET doxygen/dbclib.css)
    set(DOXYGEN_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/docs/manual)
#    doxygen_add_docs(doc_libdbc ALL
#            include/dbc doxygen
#            COMMENT "DBC Library generating API documentation with Doxygen")
endif ()


##################################################################################
#####  Packaging and installation  ###############################################
##################################################################################
include(GNUInstallDirs) # defines the CMAKE_INSTALL_* variables
# install headers
if (${CMAKE_MINOR_VERSION} GREATER_EQUAL 23)
    target_sources(dbc PUBLIC
                        FILE_SET HEADERS
                        TYPE HEADERS
                        BASE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include
                        FILES ${DBCL_HEADERS})
    install(TARGETS dbc
            EXPORT dbcTargets
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            FILE_SET HEADERS DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
else()
    install(TARGETS dbc
            EXPORT dbcTargets
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
    install(DIRECTORY ${PROJECT_SOURCE_DIR}/include/dbc DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
endif()

include(CMakePackageConfigHelpers)

set(CONFIG_VERSION_FILE "${CMAKE_CURRENT_BINARY_DIR}/dbc/dbcConfigVersion.cmake")
# generate *Version.cmake
write_basic_package_version_file(
        ${CONFIG_VERSION_FILE}
        VERSION ${PACKAGE_VERSION}
        COMPATIBILITY AnyNewerVersion)

# Export dbcTargets to file *Targets.cmake
export(EXPORT dbcTargets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/libdbc/dbcTargets.cmake"
  NAMESPACE Upstream::
)

set(ConfigPackageLocation lib/cmake/dbc)
set(CONFIG_CMAKE "${CMAKE_CURRENT_BINARY_DIR}/dbc/dbcConfig.cmake")
# create final dbc *Config.make file
# https://cmake.org/cmake/help/latest/module/CMakePackageConfigHelpers.html
configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/dbcConfig.cmake.in"
  ${CONFIG_CMAKE}
  INSTALL_DESTINATION ${ConfigPackageLocation}
)

# install *Targets.cmake
install(EXPORT dbcTargets
  FILE
    dbcTargets.cmake
  NAMESPACE
    Dbc::
  DESTINATION
    ${ConfigPackageLocation}
)

# Install *Version.cmake and *Config.cmake
install(
  FILES
    ${CONFIG_VERSION_FILE}
    ${CONFIG_CMAKE}
  DESTINATION
    ${ConfigPackageLocation}
  COMPONENT
    Devel
)
