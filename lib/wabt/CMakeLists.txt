#
# Copyright 2016 WebAssembly Community Group participants
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# WABT version : commit 0630dc53755678239d7609b61776b125221eefb8

cmake_minimum_required(VERSION 3.0.0)
project(WABT VERSION 1.0.13)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(BUILD_TOOLS "Build wabt commandline tools" ON)
option(BUILD_LIBWASM "Build libwasm" ON)
option(CODE_COVERAGE "Build with code coverage enabled" OFF)
option(WITH_EXCEPTIONS "Build with exceptions enabled" ON)
option(WERROR "Build with warnings as errors" OFF)
# WASI support is still a work in progress.
# Only a handful of syscalls are supported at this point.
option(WITH_WASI "Build WASI support via uvwasi" OFF)


if (MSVC)
  set(COMPILER_IS_CLANG 0)
  set(COMPILER_IS_MSVC 1)
elseif (CMAKE_C_COMPILER_ID MATCHES "Clang")
  set(COMPILER_IS_CLANG 1)
  set(COMPILER_IS_MSVC 0)
endif()

include(CheckIncludeFile)
include(CheckSymbolExists)

check_include_file("alloca.h" HAVE_ALLOCA_H)
check_include_file("unistd.h" HAVE_UNISTD_H)
check_symbol_exists(snprintf "stdio.h" HAVE_SNPRINTF)
check_symbol_exists(strcasecmp "strings.h" HAVE_STRCASECMP)

if (WIN32)
  check_symbol_exists(ENABLE_VIRTUAL_TERMINAL_PROCESSING "windows.h" HAVE_WIN32_VT100)
endif ()

include(CheckTypeSize)
check_type_size(ssize_t SSIZE_T)
check_type_size(size_t SIZEOF_SIZE_T)

configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/src/config.h.in
  ${CMAKE_CURRENT_SOURCE_DIR}/built/config.h
)

include_directories(chakra ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/built)

if (COMPILER_IS_MSVC)
  # disable warning C4018: signed/unsigned mismatch
  # disable warning C4056, C4756: overflow in floating-point constant arithmetic
  #   seems to not like float compare w/ HUGE_VALF; bug?
  # disable warnings C4267 and C4244: conversion/truncation from larger to smaller type.
  # disable warning C4800: implicit conversion from larger int to bool
  add_definitions(-W3 -wd4018 -wd4056 -wd4756 -wd4267 -wd4244 -wd4800 -D_CRT_SECURE_NO_WARNINGS -D_SCL_SECURE_NO_WARNINGS)

  if (WERROR)
    add_definitions(-WX)
  endif ()

  if (NOT WITH_EXCEPTIONS)
    # disable exception use in C++ library
    add_definitions(-D_HAS_EXCEPTIONS=0)
  endif ()
else ()
  # disable -Wunused-parameter: this is really common when implementing
  #   interfaces, etc.
  # disable -Wpointer-arith: this is a GCC extension, and doesn't work in MSVC.
  add_definitions(
    -Wall -Wextra -Wno-unused-parameter -Wpointer-arith -g
    -Wuninitialized
  )

  set(CMAKE_CXX_EXTENSIONS OFF)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wold-style-cast")

  if (WERROR)
    add_definitions(-Werror)
  endif ()

  if (NOT WITH_EXCEPTIONS)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions")
  endif ()

  # Need to define __STDC_*_MACROS because C99 specifies that C++ shouldn't
  # define format (e.g. PRIu64) or limit (e.g. UINT32_MAX) macros without the
  # definition, and some libcs (e.g. glibc2.17 and earlier) follow that.
  add_definitions(-D__STDC_LIMIT_MACROS=1 -D__STDC_FORMAT_MACROS=1)
  # try to get the target architecture by compiling a dummy.c file and
  # checking the architecture using the file command.
  file(WRITE ${CMAKE_CURRENT_SOURCE_DIR}/built/dummy.c "main(){}")
  try_compile(
    COMPILE_OK
    ${CMAKE_CURRENT_SOURCE_DIR}/built
    ${CMAKE_CURRENT_SOURCE_DIR}/built/dummy.c
    COPY_FILE ${CMAKE_CURRENT_SOURCE_DIR}/built/dummy
  )
  if (COMPILE_OK)
    execute_process(
      COMMAND file ${CMAKE_CURRENT_SOURCE_DIR}/built/dummy
      RESULT_VARIABLE FILE_RESULT
      OUTPUT_VARIABLE FILE_OUTPUT
      ERROR_QUIET
    )

    if (FILE_RESULT EQUAL 0)
      if (${FILE_OUTPUT} MATCHES "x86[-_]64")
        set(TARGET_ARCH "x86-64")
      elseif (${FILE_OUTPUT} MATCHES "Intel 80386")
        set(TARGET_ARCH "i386")
      elseif (${FILE_OUTPUT} MATCHES "ARM")
        set(TARGET_ARCH "ARM")
      else ()
        message(WARNING "Unknown target architecture!")
      endif ()
    else ()
      message(WARNING "Error running `file` command on dummy executable")
    endif ()
  else ()
    message(WARNING "Error compiling dummy.c file")
  endif ()

  if (TARGET_ARCH STREQUAL "i386")
    # wasm doesn't allow for x87 floating point math
    add_definitions(-msse2 -mfpmath=sse)
  endif ()
endif ()
set(USE_SANITIZER FALSE)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

add_custom_target(everything)

set(WABT_LIBRARY_SRC
  chakra/wabtapi.h
  chakra/wabtapi.cc
  src/apply-names.h
  src/apply-names.cc
  src/binary.h
  src/binary.cc
  src/binary-reader.h
  src/binary-reader.cc
  src/binary-reader-ir.h
  src/binary-reader-ir.cc
  src/binary-reader-logging.h
  src/binary-reader-logging.cc
  src/binary-writer.h
  src/binary-writer.cc
  src/binary-writer-spec.h
  src/binary-writer-spec.cc
  src/binding-hash.h
  src/binding-hash.cc
  src/color.h
  src/color.cc
  src/common.h
  src/common.cc
  src/config.h
  src/config.cc
  src/decompiler.h
  src/decompiler-ast.h
  src/decompiler-ls.h
  src/decompiler-naming.h
  src/decompiler.cc
  src/error-formatter.h
  src/error-formatter.cc
  src/expr-visitor.h
  src/expr-visitor.cc
  src/feature.h
  src/feature.cc
  src/filenames.h
  src/filenames.cc
  src/generate-names.h
  src/generate-names.cc
  src/hash-util.h
  src/hash-util.cc
  src/ir.h
  src/ir.cc
  src/ir-util.h
  src/ir-util.cc
  src/leb128.h
  src/leb128.cc
  src/lexer-source.h
  src/lexer-source.cc
  src/lexer-source-line-finder.h
  src/lexer-source-line-finder.cc
  src/literal.h
  src/literal.cc
  src/opcode.h
  src/opcode.cc
  src/opcode-code-table.h
  src/opcode-code-table.c
  src/option-parser.h
  src/option-parser.cc
  src/resolve-names.h
  src/resolve-names.cc
  src/shared-validator.h
  src/shared-validator.cc
  src/stream.h
  src/stream.cc
  src/string-view.h
  src/string-view.cc
  src/token.h
  src/token.cc
  src/tracing.h
  src/tracing.cc
  src/type.h
  src/type-checker.h
  src/type-checker.cc
  src/utf8.h
  src/utf8.cc
  src/validator.h
  src/validator.cc
  src/wast-lexer.h
  src/wast-lexer.cc
  src/wast-parser.h
  src/wast-parser.cc
  src/wat-writer.h
  src/wat-writer.cc

  # TODO(binji): Move this into its own library?
  src/interp/binary-reader-interp.h
  src/interp/binary-reader-interp.cc
  src/interp/interp.h
  src/interp/interp.cc
  src/interp/interp-inl.h
  src/interp/interp-math.h
  src/interp/interp-util.h
  src/interp/interp-util.cc
  src/interp/istream.h
  src/interp/istream.cc
)

add_library(wabt OBJECT ${WABT_LIBRARY_SRC})
