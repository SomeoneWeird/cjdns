# You may redistribute this program and/or modify it under the terms of
# the GNU General Public License as published by the Free Software Foundation,
# either version 3 of the License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
project(cjdns C)
cmake_minimum_required(VERSION 2.4)

message(${CMAKE_VERSION})

if(CMAKE_BINARY_DIR STREQUAL ${CMAKE_SOURCE_DIR} AND NOT OPENWRT)
    message( FATAL_ERROR "type: git clean -df && mkdir build && cd build && cmake .. && make" )
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}
    ${CMAKE_SOURCE_DIR}/cmake/modules)

# validation
add_definitions(-Wall -Wextra -std=c99 -Werror -pedantic)

if(WIN32)
    # ISO C does not support the ‘I64’ ms_printf length modifier
    # but PRId64 is #defined as "I64d"
    add_definitions(-Wno-error)

    # proceed with ifdef hell :(
    add_definitions(-DWIN32)
    set(PLATFORM_LIBRARIES
        ws2_32
    )

    # BufferAllocator.c:1:0: error: -fPIC ignored for target (all code is position independent)
    set(WITH_PIE OFF CACHE INTERNAL "" FORCE)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL SunOS)
    set(ILLUMOS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL Linux)
    set(LINUX TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
    set(FREEBSD TRUE)
endif()

if(CMAKE_C_COMPILER MATCHES "(.*)-gcc")
    string(REGEX REPLACE "-gcc$" "" toolchain ${CMAKE_C_COMPILER})
    string(REGEX REPLACE ".*/" "" toolchain ${toolchain})
    message("detected toolchain ${toolchain}")
    set(TOOLCHAIN ${toolchain} CACHE STRING "Toolchain for compiling (prefix for -gcc, ... binaries)")
else()
    set(TOOLCHAIN "" CACHE STRING "Toolchain for compiling (prefix for -gcc, ... binaries)")
endif()

if(TOOLCHAIN AND NOT CMAKE_C_COMPILER)
    set(CMAKE_C_COMPILER ${TOOLCHAIN}-gcc)
endif()

if(TOOLCHAIN AND NOT CMAKE_CXX_COMPILER)
    set(CMAKE_C_COMPILER ${TOOLCHAIN}-g++)
endif()

if(WIN32 AND TOOLCHAIN AND NOT CMAKE_CXX_COMPILER)
    set(CMAKE_C_COMPILER ${TOOLCHAIN}-windres)
endif()

message("c compiler: ${CMAKE_C_COMPILER}")
message("c++ compiler: ${CMAKE_CXX_COMPILER}")
if(WIN32)
    message("rc compiler: ${CMAKE_RC_COMPILER}")
endif()

# This breaks logging since loggers are passed to functions
# and only used if the log level is high.
add_definitions(-Wno-unused-parameter)

include(CheckCCompilerFlag)

# There are a number of places in Admin where read() and write() are used
# and they fail silently if anything goes wrong so the result is unused.
check_c_compiler_flag(-Wno-unused-result HAS_NO_UNUSED_RESULT)
if(HAS_NO_UNUSED_RESULT)
    add_definitions(-Wno-unused-result)
endif()

# GCC only sends a warning and not an error if it can't compile with stack canaries.
set(CMAKE_REQUIRED_FLAGS "-Werror")
check_c_compiler_flag(-fstack-protector-all HAS_F_STACK_PROTECTOR)
if(NOT HAS_F_STACK_PROTECTOR)
    message("WARNING: Stack Smashing Protector is not available on this target.")
    message("SSP is an important security measure to minimize the risk of a vulnerability.")
    message("")
else()
    add_definitions(
        # Broken GCC patch makes -fstack-protector-all not work
        # workaround is to give -fno-stack-protector first.
        # see: https://bugs.launchpad.net/ubuntu/+source/gcc-4.5/+bug/691722
        -fno-stack-protector
        -fstack-protector-all
        -Wstack-protector
    )
endif()

check_c_compiler_flag(-fcatch-undefined-behavior HAS_CATCH_UNDEFINED)
if(HAS_CATCH_UNDEFINED)
    add_definitions(-fcatch-undefined-behavior)
endif()

# __builtin_constant_p()
include(CheckCSourceCompiles)
check_c_source_compiles("int main() { return __builtin_constant_p(0);}" HAS_BUILTIN_CONSTANT_P)
if (HAS_BUILTIN_CONSTANT_P)
    add_definitions("-D HAS_BUILTIN_CONSTANT_P")
endif()

OPTION(WITH_PIE "Build a position independent executable" ON)
if (NOT "$ENV{NO_PIE}" STREQUAL "")
    set(WITH_PIE FALSE)
endif()
if(WITH_PIE)
    message("Building a position independent executable (ASLR security enabled).")
    message("NOTE: PIE executables cannot run in a debugger.")
    message("")
    add_definitions(-fPIE)
    set(PIE "-pie")
else()
    message("Building a position dependent executable (ASLR security disabled).")
    set(PIE "")

    # debugging
    add_definitions(-g)
endif()

if(NOT DEFINED APPLE AND NOT WIN32 AND NOT ILLUMOS)
    set(CMAKE_EXE_LINKER_FLAGS "${PIE} -Wl,-z,relro,-z,now,-z,noexecstack")
elseif(ILLUMOS)
    # TODO make this less ugly
    set(CMAKE_EXE_LINKER_FLAGS "${PIE} -lsocket")
else()
    # apple seems to set noexecstack by default and not support relro.
    set(CMAKE_EXE_LINKER_FLAGS "${PIE}")
endif()

# logging
SET(Log_LEVEL DEBUG CACHE STRING "Log level (KEYS, DEBUG, INFO, WARN, ERROR, CRITICAL)")
if (NOT "$ENV{Log_LEVEL}" STREQUAL "")
    set(Log_LEVEL "$ENV{Log_LEVEL}")
endif()
if (${Log_LEVEL} STREQUAL "KEYS")
    message("\n\nEXPECT TO SEE PRIVATE KEYS PRINTED IN YOUR LOGS!\n\n")
endif (${Log_LEVEL} STREQUAL "KEYS")
add_definitions("-D Log_${Log_LEVEL}")

# vrooooooom
add_definitions(-funroll-loops)

IF(NOT CMAKE_BUILD_TYPE)
# default to RelWithDebInfo (-g -O2)
    SET(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING
        "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
        FORCE)
ENDIF(NOT CMAKE_BUILD_TYPE)

OPTION(WITH_LTO "Building with link time optimization.")
if(WITH_LTO)
    add_definitions(-flto)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_C_FLAGS} -flto -fuse-linker-plugin -fwhole-program")
endif()


# Max direct connections to the switch.
SET(CJDNS_MAX_PEERS 256 CACHE STRING "maximum number of peers")
add_definitions("-D CJDNS_MAX_PEERS=${CJDNS_MAX_PEERS}")


if(NOT "$ENV{GIT_VERSION}" STREQUAL "")
    SET(GIT_VERSION "$ENV{GIT_VERSION}")
elseif(EXISTS .git_commit)
    FILE(READ .git_commit GIT_VERSION)
else()
    find_package(Git)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} log -n 1
        OUTPUT_VARIABLE GIT_LOG
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    string(REGEX MATCH "[0-9a-f]*\n" GIT_VERSION "${GIT_LOG}")
    string(REPLACE "\n" "" GIT_VERSION "${GIT_VERSION}")
endif()
# GIT_VERSION is used in dht/dhtcore/CMakeLists.txt

include_directories(${CMAKE_SOURCE_DIR})

find_package(Libevent2 REQUIRED)
include_directories(${LIBEVENT2_INCLUDE_DIRS})

find_package(NACL REQUIRED)
include_directories(${NACL_INCLUDE_DIRS})

add_subdirectory(admin)
add_subdirectory(benc)
add_subdirectory(crypto)
add_subdirectory(dht)
add_subdirectory(interface)
add_subdirectory(io)
add_subdirectory(memory)
add_subdirectory(util)
add_subdirectory(switch)
add_subdirectory(net)

add_executable(cjdroute cjdroute.c)
target_link_libraries(cjdroute
    crypto
    crypto_benchmark
    interface
    switch
    dht
    dhtcore
    cjdbenc
    cjdbenc_JsonBencSerializer
    cjdmemory
    cjdadmin
    cjdnet
    ${LIBEVENT2_LIBRARIES}
    ${PLATFORM_LIBRARIES}
)

INSTALL(TARGETS cjdroute RUNTIME DESTINATION bin)

OPTION(NO_CODESYTLE "No codestyle checks")

# check codestyle
if(NOT NO_CODESTYLE)
    set(checker ${CMAKE_SOURCE_DIR}/cmake/modules/CheckCodestyle.cmake)
    add_custom_command(OUTPUT check.codestyle
        COMMAND ${CMAKE_COMMAND}
        -DCSD=${CMAKE_SOURCE_DIR} -DSKIP=CMakeFiles -P ${checker}
    )
    add_custom_target(check_output ALL DEPENDS check.codestyle)
endif()

# Everything must be tested.
enable_testing()
add_subdirectory(test)
