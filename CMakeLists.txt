#
# Copyright 2020 Intel Corporation
# SPDX-License-Identifier: Apache 2.0
#

cmake_minimum_required(VERSION 3.10)

project(client_sdk)
set(BASE_DIR $ENV{PWD})

#set(CMAKE_VERBOSE_MAKEFILE ON)

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)
include(cmake/cli_input.cmake)
include(cmake/extension.cmake)
include(cmake/blob_path.cmake)

if (NOT(DEFINED ENV{OPENSSL3_ROOT}))
  message(FATAL_ERROR "OPENSSL3_ROOT not set")
endif()

if (NOT(DEFINED ENV{CURL_ROOT}))
  message(FATAL_ERROR "CURL_ROOT not set")
endif()

if (NOT(DEFINED ENV{SAFESTRING_ROOT}))
  message(FATAL_ERROR "SAFESTRING_ROOT not set")
endif()

if (NOT(DEFINED ENV{TINYCBOR_ROOT}))
  message(FATAL_ERROR "TINYCBOR_ROOT not set")
endif()

if( ${DA} MATCHES cse)
  if (NOT(DEFINED ENV{METEE_ROOT}))
    message(FATAL_ERROR "METEE_ROOT not set")
  endif()
endif()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${BASE_DIR}/build)
set(CMAKE_BUILD_FILES_DIRECTORY ${BASE_DIR}/build)
set(CMAKE_BUILD_DIRECTORY ${BASE_DIR}/build)
set(CMAKE_BINARY_DIR  ${BASE_DIR}/build)
set(EXECUTABLE_OUTPUT_PATH ${BASE_DIR}/build)
set(LIBRARY_OUTPUT_PATH ${BASE_DIR}/build)
set(APPLICATION_BINARY_DIR ${BASE_DIR}/build CACHE PATH "bin dir" )

client_sdk_compile_options(
  -Wswitch-default
  -Wunused-parameter
  -Wsign-compare
  -Wpedantic
  # -Werror
  -Wimplicit-function-declaration
  -Wnested-externs
  -Wmissing-prototypes
  -Wmissing-declarations
  -Wdiscarded-qualifiers
  -Wundef
  -Wincompatible-pointer-types
  -Wunused-function
  -Wunused-variable
  -Wstrict-prototypes
  -Wshadow
  -Wno-declaration-after-statement
  -Wold-style-declaration
  -Wold-style-definition
  -Wall
  )

client_sdk_compile_options(
  -fstack-protector -D_FORTIFY_SOURCE=2 -Wformat -Wformat-security -fPIE -fPIC
  )


if(${TARGET_OS} MATCHES linux)
# Safestring lib
client_sdk_include_directories(
  $ENV{OPENSSL3_ROOT}/include
  $ENV{CURL_ROOT}/include
  $ENV{SAFESTRING_ROOT}/include
  $ENV{TINYCBOR_ROOT}/src
  include
  )

if( ${DA} MATCHES cse)
  client_sdk_include_directories($ENV{METEE_ROOT}/include)
  add_subdirectory(cse)
endif()

add_subdirectory(lib)
add_subdirectory(network)
add_subdirectory(storage)
add_subdirectory(crypto)
add_subdirectory(device_modules)

endif()





if(${TARGET_OS} MATCHES linux)
  target_link_libraries(client_sdk client_sdk_interface)
  add_executable(linux-client app/main.c)
  target_sources(linux-client PRIVATE app/blob.c)

  if(${CRYPTO_HW} MATCHES true)
    target_sources(linux-client PRIVATE app/se_provisioning.c)
  endif()
  target_include_directories(linux-client PRIVATE app/include)

  # Link all the individual static libs into one single executable
  target_link_libraries(linux-client client_sdk network storage crypto)

  if( ${DA} MATCHES cse)
  if(${CSE_CLEAR} STREQUAL true)
  add_executable(cse-clear cse/clear_cse.c)
  target_include_directories(cse-clear PRIVATE cse/include)
  target_link_libraries(cse-clear metee)
  endif()
  endif()


  client_sdk_ld_options(
    -L$ENV{SAFESTRING_ROOT}/
    -l:libsafestring.a
    -L$ENV{TINYCBOR_ROOT}/lib/
    -l:libtinycbor.a
    -Wl,-z,noexecstack -Wl,-z,relro -Wl,-z,now
    -L$ENV{CURL_ROOT}/lib
    -lcurl
    )

  if (${TLS} STREQUAL openssl)
    client_sdk_ld_options(
      -L$ENV{OPENSSL3_ROOT}/lib64
      -L$ENV{OPENSSL3_ROOT}/lib
      -Wl,--no-whole-archive -lssl -lcrypto -ldl
      )
  elseif(${TLS} MATCHES mbedtls)
    client_sdk_ld_options(
      -Wl,--no-whole-archive -lmbedcrypto
      -Wl,--no-whole-archive -lmbedtls -lmbedx509
      )
  endif()

  if( ${DA} MATCHES tpm)
    client_sdk_ld_options(-ltss2-esys -ltss2-mu -ltss2-tctildr)
  endif()

  if( ${DA} MATCHES cse)
  client_sdk_ld_options(-L$ENV{METEE_ROOT} -l:libmetee.a)
  endif()

  if (${CRYPTO_HW} MATCHES true)
    client_sdk_ld_options(
      -L$ENV{CRYPTOAUTHLIB_ROOT}/lib/
      -l:libcryptoauth.so)
  endif()

endif() ### target_os linux

# CLeanup of various files
# make pristine.
add_custom_target(
  pristine
  COMMAND
  ${CMAKE_COMMAND} -P ${BASE_DIR}/cmake/pristine.cmake
  )


if (${unit-test} STREQUAL true)
  add_subdirectory(tests/unit)
endif()

if(${TARGET_OS} STREQUAL mbedos)
  add_subdirectory(mbedos)
endif()
