cmake_minimum_required(VERSION 2.8.11)
project(open62541)
# set(CMAKE_VERBOSE_MAKEFILE ON)

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/tools/cmake")
find_package(PythonInterp REQUIRED)
find_package(Git)

###########
# Version #
###########

set(OPEN62541_VER_MAJOR 0)
set(OPEN62541_VER_MINOR 2)
set(OPEN62541_VER_PATCH 0)
set(OPEN62541_VER_LABEL "-rc2") # Appended to the X.Y.Z version format. For example "-rc1" or an empty string

# Set OPEN62541_VER_COMMIT
if(GIT_FOUND)
  execute_process(COMMAND ${GIT_EXECUTABLE} describe --always --tags
                  RESULT_VARIABLE res_var OUTPUT_VARIABLE GIT_COM_ID WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
    if(${res_var} EQUAL 0)
        string(REPLACE "\n" "" OPEN62541_VER_COMMIT ${GIT_COM_ID} )
    endif()
endif()
if(NOT ${OPEN62541_VER_COMMIT} OR ${OPEN62541_VER_COMMIT} STREQUAL "")
    set(OPEN62541_VER_COMMIT "undefined")
endif()

############
# Settings #
############

# Set default build type.
if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "CMAKE_BUILD_TYPE not given; setting to 'Debug'")
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose the type of build" FORCE)
endif()

# Options
set(UA_LOGLEVEL 300 CACHE STRING "Level at which logs shall be reported")
option(UA_ENABLE_METHODCALLS "Enable the Method service set" ON)
option(UA_ENABLE_NODEMANAGEMENT "Enable dynamic addition and removal of nodes at runtime" ON)
option(UA_ENABLE_SUBSCRIPTIONS "Enable subscriptions support." ON)
option(UA_ENABLE_AMALGAMATION "Concatenate the library to a single file open62541.h/.c" OFF)
option(UA_ENABLE_COVERAGE "Enable gcov coverage" OFF)
option(BUILD_SHARED_LIBS "Enable building of shared libraries (dll/so)" OFF)

if(UA_ENABLE_COVERAGE)
  set(CMAKE_BUILD_TYPE DEBUG)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-arcs -ftest-coverage")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage -lgcov")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fprofile-arcs -ftest-coverage")
endif()

# Advanced options
option(UA_ENABLE_MULTITHREADING "Enable multithreading (experimental)" OFF)
mark_as_advanced(UA_ENABLE_MULTITHREADING)

option(UA_ENABLE_STATUSCODE_DESCRIPTIONS "Enable conversion of StatusCode to human-readable error message" ON)
mark_as_advanced(UA_ENABLE_STATUSCODE_DESCRIPTIONS)

option(UA_ENABLE_TYPENAMES "Add the type and member names to the UA_DataType structure" ON)
mark_as_advanced(UA_ENABLE_TYPENAMES)

option(UA_ENABLE_EMBEDDED_LIBC "Use a custom implementation of some libc functions that might be missing on embedded targets (e.g. string handling)." OFF)
mark_as_advanced(UA_ENABLE_EMBEDDED_LIBC)

option(UA_ENABLE_DETERMINISTIC_RNG "Do not seed the random number generator (e.g. for unit tests)." OFF)
mark_as_advanced(UA_ENABLE_DETERMINISTIC_RNG)

option(UA_ENABLE_GENERATE_NAMESPACE0 "Generate and load UA XML Namespace 0 definition (experimental)" OFF)
mark_as_advanced(UA_ENABLE_GENERATE_NAMESPACE0)

option(UA_ENABLE_EXTERNAL_NAMESPACES "Enable namespace handling by an external component (experimental)" OFF)
mark_as_advanced(UA_ENABLE_EXTERNAL_NAMESPACES)

option(UA_ENABLE_VALGRIND_UNIT_TESTS "Use Valgrind to detect memory leaks when running the unit tests" OFF)
mark_as_advanced(UA_ENABLE_VALGRIND_UNIT_TESTS)

option(UA_ENABLE_NONSTANDARD_STATELESS "Enable stateless extension (non-standard)" OFF)
mark_as_advanced(UA_ENABLE_NONSTANDARD_STATELESS)

option(UA_ENABLE_NONSTANDARD_UDP "Enable udp extension (non-standard)" OFF)
mark_as_advanced(UA_ENABLE_NONSTANDARD_UDP)
if(UA_ENABLE_NONSTANDARD_UDP)
  set(UA_ENABLE_NONSTANDARD_STATELESS ON)
endif()

# Build Targets
option(UA_BUILD_EXAMPLES "Build example servers and clients" OFF)
option(UA_BUILD_UNIT_TESTS "Build the unit tests" OFF)
option(UA_BUILD_EXAMPLES_NODESET_COMPILER "Generate an OPC UA information model from a nodeset XML (experimental)" OFF)

# Advanced Build Targets
option(UA_BUILD_SELFSIGNED_CERTIFICATE "Generate self-signed certificate" OFF)
mark_as_advanced(UA_BUILD_SELFSIGNED_CERTIFICATE)

# Building shared libs (dll, so). This option is written into ua_config.h.
set(UA_DYNAMIC_LINKING OFF)
if(BUILD_SHARED_LIBS)
  set(UA_DYNAMIC_LINKING ON)
endif()

# Force compilation with as C++
option(UA_COMPILE_AS_CXX "Force compilation with a C++ compiler" OFF)
mark_as_advanced(UA_COMPILE_AS_CXX)

#####################
# Compiler Settings #
#####################

list(APPEND open62541_LIBRARIES "") # Collect libraries

if(CMAKE_COMPILER_IS_GNUCC OR "x${CMAKE_C_COMPILER_ID}" STREQUAL "xClang")
    # Compiler
    add_definitions(-std=c99 -pipe -Wall -Wextra -Werror -Wformat -Wno-unused-parameter
                    -Wno-unused-function -Wno-unused-label -Wpointer-arith -Wreturn-type -Wsign-compare
                    -Wmultichar -Wstrict-overflow -Wcast-qual -Wmissing-prototypes -Wstrict-prototypes
                    -Winit-self -Wuninitialized -Wformat-security -Wformat-nonliteral)
    if(NOT WIN32 AND NOT CYGWIN)
        add_definitions(-Wshadow -Wconversion -fvisibility=hidden -fPIC)
    endif()

    # Linker
    set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "") # cmake sets -rdynamic by default

    # Debug
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        # add_definitions(-fsanitize=address)
        # list(APPEND open62541_LIBRARIES asan)
        # add_definitions(-fsanitize=undefined)
        # list(APPEND open62541_LIBRARIES ubsan)
    endif()

    # Strip release builds
    if(CMAKE_BUILD_TYPE STREQUAL "MinSizeRel" OR CMAKE_BUILD_TYPE STREQUAL "Release")
        add_definitions(-ffunction-sections -fdata-sections -fno-stack-protector -fno-unwind-tables
                        -fno-asynchronous-unwind-tables -fno-math-errno -fmerge-all-constants -fno-ident)
        set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -s")
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -s")
        if(APPLE)
            set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -Wl,-dead_strip")
            set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-dead_strip")
        else()
            set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -Wl,--gc-sections")
            set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--gc-sections")
        endif()
        if(NOT WIN32 AND NOT CYGWIN AND NOT APPLE)
            # these settings reduce the binary size by ~2kb
            set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -Wl,-z,norelro -Wl,--hash-style=gnu -Wl,--build-id=none")
        endif()
    endif()
endif()

if(APPLE)
    set(CMAKE_MACOSX_RPATH 1)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_DARWIN_C_SOURCE=1")
endif()

if(MSVC)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /W3 /WX") # Compiler warnings, error on warning
  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MT")
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /MTd")
endif()

#########################
# Generate Main Library #
#########################

file(MAKE_DIRECTORY "${PROJECT_BINARY_DIR}/src_generated")
configure_file("include/ua_config.h.in" "${PROJECT_BINARY_DIR}/src_generated/ua_config.h")
include_directories(${PROJECT_BINARY_DIR}/src_generated
                    ${PROJECT_SOURCE_DIR}/include
                    ${PROJECT_SOURCE_DIR}/deps)

set(exported_headers ${PROJECT_BINARY_DIR}/src_generated/ua_config.h
                     ${PROJECT_SOURCE_DIR}/deps/ms_stdint.h
                     ${PROJECT_SOURCE_DIR}/include/ua_constants.h
                     ${PROJECT_SOURCE_DIR}/include/ua_types.h
                     ${PROJECT_BINARY_DIR}/src_generated/ua_nodeids.h
                     ${PROJECT_BINARY_DIR}/src_generated/ua_types_generated.h
                     ${PROJECT_BINARY_DIR}/src_generated/ua_types_generated_handling.h
                     ${PROJECT_SOURCE_DIR}/include/ua_connection.h
                     ${PROJECT_SOURCE_DIR}/include/ua_job.h
                     ${PROJECT_SOURCE_DIR}/include/ua_log.h
                     ${PROJECT_SOURCE_DIR}/include/ua_server.h
                     ${PROJECT_SOURCE_DIR}/include/ua_server_external_ns.h
                     ${PROJECT_SOURCE_DIR}/include/ua_client.h
                     ${PROJECT_SOURCE_DIR}/include/ua_client_highlevel.h
                     ${PROJECT_SOURCE_DIR}/plugins/ua_network_tcp.h
                     ${PROJECT_SOURCE_DIR}/plugins/ua_log_stdout.h
                     ${PROJECT_SOURCE_DIR}/plugins/ua_config_standard.h)
set(internal_headers ${PROJECT_SOURCE_DIR}/deps/queue.h
                     ${PROJECT_SOURCE_DIR}/deps/pcg_basic.h
                     ${PROJECT_SOURCE_DIR}/deps/libc_time.h
                     ${PROJECT_SOURCE_DIR}/src/ua_util.h
                     ${PROJECT_SOURCE_DIR}/src/ua_types_encoding_binary.h
                     ${PROJECT_BINARY_DIR}/src_generated/ua_types_generated_encoding_binary.h
                     ${PROJECT_BINARY_DIR}/src_generated/ua_transport_generated.h
                     ${PROJECT_BINARY_DIR}/src_generated/ua_transport_generated_handling.h
                     ${PROJECT_BINARY_DIR}/src_generated/ua_transport_generated_encoding_binary.h
                     ${PROJECT_SOURCE_DIR}/src/ua_connection_internal.h
                     ${PROJECT_SOURCE_DIR}/src/ua_securechannel.h
                     ${PROJECT_SOURCE_DIR}/src/server/ua_nodes.h
                     ${PROJECT_SOURCE_DIR}/src/ua_session.h
                     ${PROJECT_SOURCE_DIR}/src/server/ua_subscription.h
                     ${PROJECT_SOURCE_DIR}/src/server/ua_nodestore.h
                     ${PROJECT_SOURCE_DIR}/src/server/ua_session_manager.h
                     ${PROJECT_SOURCE_DIR}/src/server/ua_securechannel_manager.h
                     ${PROJECT_SOURCE_DIR}/src/server/ua_server_internal.h
                     ${PROJECT_SOURCE_DIR}/src/server/ua_services.h
                     ${PROJECT_SOURCE_DIR}/src/client/ua_client_internal.h)
set(lib_sources ${PROJECT_SOURCE_DIR}/src/ua_types.c
                ${PROJECT_SOURCE_DIR}/src/ua_types_encoding_binary.c
                ${PROJECT_BINARY_DIR}/src_generated/ua_types_generated.c
                ${PROJECT_BINARY_DIR}/src_generated/ua_transport_generated.c
                ${PROJECT_SOURCE_DIR}/src/ua_connection.c
                ${PROJECT_SOURCE_DIR}/src/ua_securechannel.c
                ${PROJECT_SOURCE_DIR}/src/ua_session.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_server.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_server_binary.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_server_utils.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_server_worker.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_securechannel_manager.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_session_manager.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_nodes.c
                # nodestores
                ${PROJECT_SOURCE_DIR}/src/server/ua_nodestore.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_nodestore_concurrent.c
                # services
                ${PROJECT_SOURCE_DIR}/src/server/ua_services_discovery.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_services_securechannel.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_services_session.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_services_attribute.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_services_nodemanagement.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_services_view.c
                # method call
                ${PROJECT_SOURCE_DIR}/src/server/ua_services_call.c
                # subscriptions
                ${PROJECT_SOURCE_DIR}/src/server/ua_subscription.c
                ${PROJECT_SOURCE_DIR}/src/server/ua_services_subscription.c
                # client
                ${PROJECT_SOURCE_DIR}/src/client/ua_client.c
                ${PROJECT_SOURCE_DIR}/src/client/ua_client_highlevel.c
                ${PROJECT_SOURCE_DIR}/src/client/ua_client_highlevel_subscriptions.c
                # plugins and dependencies
                ${PROJECT_SOURCE_DIR}/plugins/ua_network_tcp.c
                ${PROJECT_SOURCE_DIR}/plugins/ua_clock.c
                ${PROJECT_SOURCE_DIR}/plugins/ua_log_stdout.c
                ${PROJECT_SOURCE_DIR}/plugins/ua_config_standard.c
                ${PROJECT_SOURCE_DIR}/deps/libc_time.c
                ${PROJECT_SOURCE_DIR}/deps/pcg_basic.c)
                ##TODO: make client stuff optional

if(UA_ENABLE_EMBEDDED_LIBC)
  list(APPEND lib_sources ${PROJECT_SOURCE_DIR}/deps/libc_string.c)
endif()

if(UA_ENABLE_GENERATE_NAMESPACE0)
  set(GENERATE_NAMESPACE0_FILE "Opc.Ua.NodeSet2.xml" CACHE STRING "Namespace definition XML file")
  set_property(CACHE GENERATE_NAMESPACE0_FILE PROPERTY STRINGS Opc.Ua.NodeSet2.xml Opc.Ua.NodeSet2.Minimal.xml)
  list(APPEND internal_headers ${PROJECT_BINARY_DIR}/src_generated/ua_namespaceinit_generated.h)
  list(APPEND lib_sources ${PROJECT_BINARY_DIR}/src_generated/ua_namespaceinit_generated.c)
endif()

if(UA_ENABLE_NONSTANDARD_UDP)
    list(APPEND exported_headers ${PROJECT_SOURCE_DIR}/plugins/ua_network_udp.h)
endif()

#########################
# Generate source files #
#########################

# standard data types
add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/src_generated/ua_types_generated.c
                          ${PROJECT_BINARY_DIR}/src_generated/ua_types_generated.h
                          ${PROJECT_BINARY_DIR}/src_generated/ua_types_generated_handling.h
                          ${PROJECT_BINARY_DIR}/src_generated/ua_types_generated_encoding_binary.h
                   PRE_BUILD
                   COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/tools/generate_datatypes.py
                           --typedescriptions ${PROJECT_SOURCE_DIR}/tools/schema/NodeIds.csv
                           --selected_types=${PROJECT_SOURCE_DIR}/tools/schema/datatypes_minimal.txt
                           ${PROJECT_SOURCE_DIR}/tools/schema/Opc.Ua.Types.bsd ${PROJECT_BINARY_DIR}/src_generated/ua_types
                   DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/tools/generate_datatypes.py
                           ${PROJECT_SOURCE_DIR}/tools/schema/datatypes_minimal.txt
                           ${CMAKE_CURRENT_SOURCE_DIR}/tools/schema/Opc.Ua.Types.bsd
                           ${CMAKE_CURRENT_SOURCE_DIR}/tools/schema/NodeIds.csv)

# transport data types
add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/src_generated/ua_transport_generated.c
                          ${PROJECT_BINARY_DIR}/src_generated/ua_transport_generated.h
                          ${PROJECT_BINARY_DIR}/src_generated/ua_transport_generated_handling.h
                          ${PROJECT_BINARY_DIR}/src_generated/ua_transport_generated_encoding_binary.h
                   PRE_BUILD
                   COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/tools/generate_datatypes.py
                           --namespace=1 --selected_types=${PROJECT_SOURCE_DIR}/tools/schema/datatypes_transport.txt
                           ${PROJECT_SOURCE_DIR}/tools/schema/Opc.Ua.Types.bsd
                           ${PROJECT_SOURCE_DIR}/tools/schema/Custom.Opc.Ua.Transport.bsd
                           ${PROJECT_BINARY_DIR}/src_generated/ua_transport
                   DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/tools/generate_datatypes.py
                           ${PROJECT_SOURCE_DIR}/tools/schema/datatypes_transport.txt
                           ${CMAKE_CURRENT_SOURCE_DIR}/tools/schema/Custom.Opc.Ua.Transport.bsd)

# nodeids
add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/src_generated/ua_nodeids.h
                   PRE_BUILD
                   COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/tools/generate_nodeids.py
                           ${PROJECT_SOURCE_DIR}/tools/schema/NodeIds.csv ${PROJECT_BINARY_DIR}/src_generated/ua_nodeids
                   DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/tools/generate_nodeids.py
                           ${CMAKE_CURRENT_SOURCE_DIR}/tools/schema/NodeIds.csv)

# statuscode explanation
add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/src_generated/ua_statuscode_descriptions.c
        PRE_BUILD
        COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/tools/generate_statuscode_descriptions.py
        ${PROJECT_SOURCE_DIR}/tools/schema/Opc.Ua.StatusCodes.csv ${PROJECT_BINARY_DIR}/src_generated/ua_statuscode_descriptions
        DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/tools/generate_statuscode_descriptions.py
        ${CMAKE_CURRENT_SOURCE_DIR}/tools/schema/Opc.Ua.StatusCodes.csv)
list(APPEND lib_sources ${PROJECT_BINARY_DIR}/src_generated/ua_statuscode_descriptions.c)

# generated namespace 0
add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/src_generated/ua_namespaceinit_generated.c
                          ${PROJECT_BINARY_DIR}/src_generated/ua_namespaceinit_generated.h
                   PRE_BUILD
                   COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/generate_open62541CCode.py
                           -i ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/NodeID_AssumeExternal.txt
                           -s description -b ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/NodeID_Blacklist.txt
                           ${PROJECT_SOURCE_DIR}/tools/schema/namespace0/${GENERATE_NAMESPACE0_FILE}
                           ${PROJECT_BINARY_DIR}/src_generated/ua_namespaceinit_generated
                   DEPENDS ${PROJECT_SOURCE_DIR}/tools/schema/namespace0/${GENERATE_NAMESPACE0_FILE}
                           ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/generate_open62541CCode.py
                           ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/open62541_MacroHelper.py
                           ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/ua_builtin_types.py
                           ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/ua_constants.py
                           ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/ua_namespace.py
                           ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/ua_node_types.py)

# single-file release
add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/open62541.h
                   PRE_BUILD
                   COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/tools/amalgamate.py
                           ${OPEN62541_VER_COMMIT} ${CMAKE_CURRENT_BINARY_DIR}/open62541.h ${exported_headers}
                   DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/tools/amalgamate.py
                           ${exported_headers}
                           ${internal_headers})

add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/open62541.c
                   PRE_BUILD
                   COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/tools/amalgamate.py
                           ${OPEN62541_VER_COMMIT} ${CMAKE_CURRENT_BINARY_DIR}/open62541.c ${internal_headers} ${lib_sources}
                   DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/tools/amalgamate.py ${internal_headers} ${lib_sources})

ADD_CUSTOM_TARGET(open625451_amalgamation DEPENDS ${PROJECT_BINARY_DIR}/open62541.h
                                                  ${PROJECT_BINARY_DIR}/open62541.c)

# example information model from nodeset xml
add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/src_generated/nodeset.h ${PROJECT_BINARY_DIR}/src_generated/nodeset.c
                  PRE_BUILD
                  COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/generate_open62541CCode.py
                                                -i ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/NodeID_Blacklist_FullNS0.txt
                                                ${PROJECT_SOURCE_DIR}/tools/schema/namespace0/Opc.Ua.NodeSet2.xml
                                                ${PROJECT_SOURCE_DIR}/examples/server_nodeset.xml
                                                ${PROJECT_BINARY_DIR}/src_generated/nodeset
                  DEPENDS ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/generate_open62541CCode.py
                          ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/open62541_MacroHelper.py
                          ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/ua_builtin_types.py
                          ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/ua_constants.py
                          ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/ua_namespace.py
                          ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/ua_node_types.py
                          ${PROJECT_SOURCE_DIR}/tools/pyUANamespace/NodeID_Blacklist_FullNS0.txt
                          ${PROJECT_SOURCE_DIR}/examples/server_nodeset.xml)

#####################
# Build the Library #
#####################

if(UA_ENABLE_AMALGAMATION)
    add_library(open62541-object OBJECT ${PROJECT_BINARY_DIR}/open62541.c ${PROJECT_BINARY_DIR}/open62541.h)
    target_include_directories(open62541-object PRIVATE ${PROJECT_BINARY_DIR})
    if(UA_COMPILE_AS_CXX)
        set_source_files_properties(${PROJECT_BINARY_DIR}/open62541.c PROPERTIES LANGUAGE CXX)
    endif()
else()
    add_definitions(-DUA_NO_AMALGAMATION)
    add_library(open62541-object OBJECT ${lib_sources} ${internal_headers} ${exported_headers})
    target_include_directories(open62541-object PRIVATE ${PROJECT_SOURCE_DIR}/src ${PROJECT_SOURCE_DIR}/plugins)

    if(UA_COMPILE_AS_CXX)
        set_source_files_properties(${lib_sources} PROPERTIES LANGUAGE CXX)
    endif()
endif()
add_library(open62541 $<TARGET_OBJECTS:open62541-object>)
target_link_libraries(open62541 ${open62541_LIBRARIES})

target_compile_definitions(open62541-object PRIVATE -DUA_DYNAMIC_LINKING_EXPORT)
target_compile_definitions(open62541 PRIVATE -DUA_DYNAMIC_LINKING_EXPORT)

# Generate properly versioned shared library links on Linux
SET_TARGET_PROPERTIES(open62541 PROPERTIES SOVERSION 0 VERSION "${OPEN62541_VER_MAJOR}.${OPEN62541_VER_MINOR}.${OPEN62541_VER_PATCH}")

if(WIN32)
    target_link_libraries(open62541 ws2_32)
endif()

##########################
# Build Selected Targets #
##########################

# always include, builds with make doc
add_subdirectory(doc)

if(UA_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

if(UA_BUILD_UNIT_TESTS)
    target_compile_definitions(open62541-object PRIVATE UA_BUILD_UNIT_TESTS) # Export some internal functions
    enable_testing()
    add_subdirectory(tests)
endif()

if(UA_BUILD_EXAMPLES_NODESET_COMPILER)
  add_custom_target(generate_informationmodel ALL
                    DEPENDS ${PROJECT_BINARY_DIR}/src_generated/nodeset.h ${PROJECT_BINARY_DIR}/src_generated/nodeset.c)
endif()

############################
# Linting run (clang-tidy) #
############################

find_package(ClangTools)
add_custom_target(lint ${CLANG_TIDY_PROGRAM}
                  ${lib_sources}
                  -checks=cert-*,performance-*,readability-*,-readability-braces-around-statements
                  -warnings-as-errors=cert-*,performance-*,readability-*,-readability-braces-around-statements
                  --
                  -std=c99
                  -I${PROJECT_SOURCE_DIR}/include
                  -I${PROJECT_SOURCE_DIR}/plugins
                  -I${PROJECT_SOURCE_DIR}/deps
                  -I${PROJECT_SOURCE_DIR}/src
                  -I${PROJECT_SOURCE_DIR}/src/server
                  -I${PROJECT_SOURCE_DIR}/src/client
                  -I${PROJECT_BINARY_DIR}/src_generated
                  DEPENDS ${lib_sources}
                  COMMENT "Run clang-tidy on the library")
add_dependencies(lint open62541)

##########################
# Installation           #
##########################
# invoke via `make install`
# specify install location with `-DCMAKE_INSTALL_PREFIX=xyz`
# Enable shared library with `-DBUILD_SHARED_LIBS=ON`

# export library (either static or shared depending on BUILD_SHARED_LIBS)
install(TARGETS open62541
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)
# export amalgamated header open62541.h which is generated due to build of open62541-object
install(FILES ${PROJECT_BINARY_DIR}/open62541.h DESTINATION include)

##########################
# Packaging (DEB/RPM)    #
##########################
# invoke via `make package`

set(CPACK_GENERATOR "TGZ;DEB;RPM")
set(CPACK_PACKAGE_VENDOR "open62541 team")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "OPC UA implementation")
set(CPACK_PACKAGE_DESCRIPTION "open62541 is a C-based library (linking with C++ projects is possible) with all necessary tools to implement dedicated OPC UA clients and servers, or to integrate OPC UA-based communication into existing applications.")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
set(CPACK_PACKAGE_VERSION_MAJOR "0")
set(CPACK_PACKAGE_VERSION_MINOR "2")
set(CPACK_PACKAGE_VERSION_PATCH "0")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "open62541 team") #required

include(CPack)
