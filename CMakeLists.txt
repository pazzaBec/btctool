cmake_minimum_required (VERSION 2.6)

project (TOOL)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")

###################################### Options ######################################
message("") # empty line
message("------------------------------- Options -------------------------------")

#
# System Type:
#
MESSAGE(STATUS "Current platform: ${CMAKE_SYSTEM_NAME}")
MESSAGE(STATUS "operation system is ${CMAKE_SYSTEM}")

#
# Build Type: -DCMAKE_BUILD_TYPE=Debug|Release
# Default: Release
#
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()
message("-- Build Type: ${CMAKE_BUILD_TYPE} (-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})")
IF(CMAKE_BUILD_TYPE MATCHES Debug)
  set(CMAKE_CXX_FLAGS "-g -O0 -Wall")
ELSEIF(CMAKE_BUILD_TYPE MATCHES Release)
  set(CMAKE_CXX_FLAGS "-g -O2 -Wall")
ELSE()
  set(CMAKE_CXX_FLAGS "-g -Wall")
ENDIF()

# compiler options
set(CMAKE_CXX_COMPILER "g++")
set(CMAKE_C_COMPILER "gcc")
set(CMAKE_CXX_COMPILER_ARG1 "-std=c++0x")

#
#Environment checks
#
IF(APPLE)
    add_definitions( "-DMAC_OSX" )
ELSE()
    #add_definitions("-rdynamic")
ENDIF()

add_definitions("-pthread")

# add a macro that needed by chain's source code
add_definitions(-DHAVE_CONFIG_H)

#
#Mysql checks
#
execute_process(COMMAND mysql_config --libs_r OUTPUT_VARIABLE MYSQL_LIB OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND mysql_config --include OUTPUT_VARIABLE MYSQL_INCLUDE OUTPUT_STRIP_TRAILING_WHITESPACE)
message("-- MySQL include: ${MYSQL_INCLUDE}")
message("-- MySQL library: ${MYSQL_LIB}")
if(NOT MYSQL_LIB)
  message(STATUS "MySQL library not found!")
endif()

if(NOT MYSQL_INCLUDE)
  message(STATUS "MySQL header not found!")
endif()

if(MYSQL_LIB AND MYSQL_INCLUDE)
  set(MYSQL "use")
  add_definitions("-DMYSQL_ENABLE")
endif()

#
# The number of concurrent jobs when compiling a third-party library
# Example: -DJOBS=2
# Default: 1
#
if(JOBS)
  message("-- Concurrent Jobs: ${JOBS} (-DJOBS=${JOBS})")
  set(MAKE_JOBS "-j${JOBS}") 
else()
  message("-- Concurrent Jobs: 1 (-DJOBS=1)")
  set(MAKE_JOBS "") 
endif()

#
# Chain Type
#
set(SUPPORTED_CHAIN_TYPES "BTC|QTM|UT")
if(NOT CHAIN_TYPE)
  message(FATAL_ERROR "Chain type not defined! (-DCHAIN_TYPE=${SUPPORTED_CHAIN_TYPES})")
endif()

# Bitcoin
if (CHAIN_TYPE STREQUAL "BTC")
  set(CHAIN_NAME "Bitcoin Core")

# Qtum
elseif(CHAIN_TYPE STREQUAL "QTM")
  set(CHAIN_NAME "Qtum core")

# ULord
elseif(CHAIN_TYPE STREQUAL "UT")
  set(CHAIN_NAME "Ulord core")

# Unknown
else()
  message("Unknown chain type! (-DCHAIN_TYPE=${CHAIN_TYPE})")
  message(FATAL_ERROR "Supported chain type: ${SUPPORTED_CHAIN_TYPES}")
endif()

###################################### Libs ######################################
message("") # empty line
message("------------------------------- Libs -------------------------------")

#
# chain's libs
#

#
#Ulord
#
if(CHAIN_TYPE STREQUAL "UT")
  set(CHAINCORE_LIBRARIES ${CHAINCORE_LIBRARIES}
      "${CHAIN_SRC_ROOT}/src/libbitcoin_server.a"
      "${CHAIN_SRC_ROOT}/src/libbitcoin_wallet.a"
      "${CHAIN_SRC_ROOT}/src/libbitcoin_common.a"
      "${CHAIN_SRC_ROOT}/src/libbitcoin_util.a"
      "${CHAIN_SRC_ROOT}/src/libbitcoin_zmq.a"
      "${CHAIN_SRC_ROOT}/src/hello/libbitcoin_hello.a"
      "${CHAIN_SRC_ROOT}/src/crypto/libbitcoin_crypto.a"
    )

#
#Qtum & BTC
#
else()
  set(CHAINCORE_LIBRARIES ${CHAINCORE_LIBRARIES}
    "${CHAIN_SRC_ROOT}/src/libbitcoin_common.a"
    "${CHAIN_SRC_ROOT}/src/libbitcoin_consensus.a"
    "${CHAIN_SRC_ROOT}/src/libbitcoin_util.a"
    "${CHAIN_SRC_ROOT}/src/crypto/libbitcoin_crypto_base.a"
    "${CHAIN_SRC_ROOT}/src/crypto/libbitcoin_crypto_avx2.a"
    "${CHAIN_SRC_ROOT}/src/crypto/libbitcoin_crypto_shani.a"
    "${CHAIN_SRC_ROOT}/src/crypto/libbitcoin_crypto_sse41.a"
    )
endif()

list(GET CHAINCORE_LIBRARIES 0 CORE_LIBRARIE)
if (NOT EXISTS ${CORE_LIBRARIE})
  message(STATUS "building ${CHAIN_NAME}...")
  execute_process(WORKING_DIRECTORY "${CHAIN_SRC_ROOT}" COMMAND ./autogen.sh)
  execute_process(WORKING_DIRECTORY "${CHAIN_SRC_ROOT}" COMMAND ./configure --disable-tests --disable-bench)
  execute_process(WORKING_DIRECTORY "${CHAIN_SRC_ROOT}" COMMAND make ${MAKE_JOBS})
endif()

foreach(CORE_LIBRARIE ${CHAINCORE_LIBRARIES})
  if (NOT EXISTS ${CORE_LIBRARIE})
    message(FATAL_ERROR "${CORE_LIBRARIE} not exists!")
  endif()
endforeach()

#
# libsecp256k1
#
set(secp256k1_DIR "${CHAIN_SRC_ROOT}/src/secp256k1")
set(secp256k1_LIBRARIES "${secp256k1_DIR}/.libs/libsecp256k1.a")
if (NOT EXISTS ${secp256k1_LIBRARIES})
  message(STATUS "build secp256k1...")
  execute_process(WORKING_DIRECTORY "${secp256k1_DIR}" COMMAND ./autogen.sh)
  execute_process(WORKING_DIRECTORY "${secp256k1_DIR}" COMMAND ./configure --enable-module-recovery)
  execute_process(WORKING_DIRECTORY "${secp256k1_DIR}" COMMAND make ${MAKE_JOBS})
endif()
if (NOT EXISTS ${secp256k1_LIBRARIES})
  message(FATAL_ERROR "build secp256k1 failed!")
endif()

#
# libunivalue
#
set(univalue_DIR "${CHAIN_SRC_ROOT}/src/univalue")
set(univalue_LIBRARIES "${univalue_DIR}/.libs/libunivalue.a")
if (NOT EXISTS ${univalue_LIBRARIES})
  message(STATUS "build univalue...")
  execute_process(WORKING_DIRECTORY "${univalue_DIR}" COMMAND ./autogen.sh)
  execute_process(WORKING_DIRECTORY "${univalue_DIR}" COMMAND ./configure --enable-module-recovery)
  execute_process(WORKING_DIRECTORY "${univalue_DIR}" COMMAND make ${MAKE_JOBS})
endif()
if (NOT EXISTS ${univalue_LIBRARIES})
  message(FATAL_ERROR "build univalue failed!")
endif()

#
# libleveldb
#
set(leveldb_DIR "${CHAIN_SRC_ROOT}/src/leveldb")
set(leveldb_LIBRARIES "${leveldb_DIR}/libleveldb.a" "${leveldb_DIR}/libmemenv.a")
foreach(LEVELDB_LIBRARIE ${leveldb_LIBRARIES})
  if (NOT EXISTS ${LEVELDB_LIBRARIE})
    message(FATAL_ERROR "${LEVELDB_LIBRARIE} not exists!")
  endif()
endforeach()

#
#Third libs
#
find_package(OpenSSL)
if(NOT OPENSSL_FOUND)
  message(FATAL_ERROR "OpenSSL not found!")
endif()

find_package(libzmq)
if(NOT LIBZMQ_FOUND)
  message(FATAL_ERROR "libzmq not found!")
endif()

find_package(Glog)
if(NOT GLOG_FOUND)
  message(FATAL_ERROR "Glog not found!")
endif()

find_package(LibEvent)
if(NOT LibEvent_FOUND)
  message(FATAL_ERROR "libevent2 not found!")
endif()

find_package(Boost 1.36.0 COMPONENTS log system filesystem program_options thread regex timer chrono serialization)
if(NOT Boost_FOUND)
  message(FATAL_ERROR "Boost not found!")
endif()

find_package(LibPthread)
if(NOT PTHREAD_FOUND)
  message(FATAL_ERROR "libpthread not found!")
endif()

find_package(BDB)
if(NOT BDB_FOUND)
  message(FATAL_ERROR "Berkeley DB not found!")
endif()

message("") # add an empty line

###################################### Targets ######################################
message("") # empty line
message("------------------------------- Targets -------------------------------")

#
#Include path
#
include_directories(common
                    ${CHAIN_SRC_ROOT}/src
                    ${CHAIN_SRC_ROOT}/src/config
                    ${CHAIN_SRC_ROOT}/src/wallet
                    ${CHAIN_SRC_ROOT}/src/crypto
                    ${CHAIN_SRC_ROOT}/src/hello
                    ${secp256k1_DIR}/include
                    ${univalue_DIR}/include
                    ${leveldb_DIR}/include
                    ${OPENSSL_INCLUDE_DIR}
                    ${Boost_INCLUDE_DIRS}
                    ${LIBEVENT_INCLUDE_DIR}
                    ${LIBZMQ_INCLUDE_DIR}
                    ${BDB_INCLUDE_DIR}
                    ${GLOG_INCLUDE_DIRS}
                    )

if(CHAIN_TYPE STREQUAL "QTM")
  include_directories(${CHAIN_SRC_ROOT}/src/cpp-ethereum)
endif()

set(THIRD_LIBRARIES ${CHAINCORE_LIBRARIES}
                    ${OPENSSL_CRYPTO_LIBRARY}
                    ${OPENSSL_SSL_LIBRARY}
                    ${univalue_LIBRARIES}
                    ${leveldb_LIBRARIES}
                    ${Boost_LIBRARIES}
                    ${LIBZMQ_LIBRARIES}
                    ${BDB_LIBRARYS}
                    ${LIBEVENT_LIB}
                    ${LIBEVENT_PTHREADS_LIB}
                    ${secp256k1_LIBRARIES}
                    ${PTHREAD_LIBRARIES}
                    ${GLOG_LIBRARIES}
                    )

if(MYSQL)
  include_directories(${MYSQL_INCLUDE})
  list(APPEND THIRD_LIBRARIES ${MYSQL_LIB})
endif()

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)

# Bitcoin
if (CHAIN_TYPE STREQUAL "BTC")
  add_subdirectory(btc)

# Qtum
elseif(CHAIN_TYPE STREQUAL "QTM")
  add_subdirectory(qtum)

# ULord
elseif(CHAIN_TYPE STREQUAL "UT")
  add_subdirectory(ulord)
endif()

message("------------------------------- tool end ------------------------------")
message("") # empty line