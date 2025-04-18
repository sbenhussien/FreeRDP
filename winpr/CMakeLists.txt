# WinPR: Windows Portable Runtime
# winpr cmake build script
#
# Copyright 2012 Marc-Andre Moreau <marcandre.moreau@gmail.com>
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

if (NOT FREERDP_UNIFIED_BUILD)
	cmake_minimum_required(VERSION 3.4)
	project(WinPR LANGUAGES C)

	set(CMAKE_COLOR_MAKEFILE ON)
	option(WITH_LIBRARY_VERSIONING "Use library version triplet" ON)

	# Default to build shared libs
	option(BUILD_SHARED_LIBS "Build shared libraries" ON)
    option(EXPORT_ALL_SYMBOLS "Export all symbols form library" OFF)
    if(CMAKE_COMPILER_IS_GNUCC)
        if(NOT EXPORT_ALL_SYMBOLS)
            message(STATUS "GCC default symbol visibility: hidden")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
        endif()
    endif()
endif()

set(DEFAULT_DEBUG_OPTION OFF)

if (WIN32 AND NOT UWP)
	set(NATIVE_SSPI ON)
endif()
option(WITH_VERBOSE_WINPR_ASSERT "Compile with verbose WINPR_ASSERT." ON)
option(WITH_WINPR_TOOLS "Build WinPR helper binaries" ON)
option(WITH_WINPR_DEPRECATED "Build WinPR deprecated symbols" OFF)
option(WITH_DEBUG_THREADS "Print thread debug messages, enables handle dump" ${DEFAULT_DEBUG_OPTION})
option(WITH_DEBUG_EVENTS "Print event debug messages, enables handle dump" ${DEFAULT_DEBUG_OPTION})
option(WITH_DEBUG_SYMBOLS "Pack debug symbols to installer" OFF)
option(WITH_NATIVE_SSPI "Use native SSPI modules" ${NATIVE_SSPI})
option(WITH_SMARTCARD_INSPECT "Enable SmartCard API Inspector" OFF)
option(WITH_DEBUG_MUTEX "Print mutex debug messages" ${DEFAULT_DEBUG_OPTION})
option(WITH_ICU "Use ICU for unicode conversion" OFF)

option(WITH_DEBUG_NTLM "Print NTLM debug messages" ${DEFAULT_DEBUG_OPTION})
if(WITH_DEBUG_NTLM)
    message(WARNING "WITH_DEBUG_NTLM=ON, the build might leak sensitive information, do not use with release builds!")
endif()

option(WITH_DEBUG_NLA "Print authentication related debug messages." ${DEFAULT_DEBUG_OPTION})
if(WITH_DEBUG_NLA)
    message(WARNING "WITH_DEBUG_NLA=ON, the build might leak sensitive information, do not use with release builds!")
endif()

if (WITH_WINPR_DEPRECATED)
	add_definitions(-DWITH_WINPR_DEPRECATED)
endif()

if (WITH_VERBOSE_WINPR_ASSERT)
    add_definitions(-DWITH_VERBOSE_WINPR_ASSERT)
endif()


# Include cmake modules
include(CheckIncludeFiles)
include(CheckLibraryExists)
include(CheckSymbolExists)
include(CheckStructHasMember)
include(FindPkgConfig)
include(TestBigEndian)

# Include our extra modules
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../cmake/)

# Check for cmake compatibility (enable/disable features)
include(CheckCmakeCompat)
include(FindFeature)
include(ComplexLibrary)
include(FeatureSummary)
include(CheckCCompilerFlag)
include(GNUInstallDirsWrapper)
include(InstallFreeRDPMan)
include(SetFreeRDPCMakeInstallDir)
include(CMakePackageConfigHelpers)

if (NOT WIN32)
    add_definitions(-DWINPR_CRITICAL_SECTION_DISABLE_SPINCOUNT)
endif()

# Soname versioning
set(RAW_VERSION_STRING "3.0.0-dev")
if(EXISTS "${PROJECT_SOURCE_DIR}/.source_tag")
	file(READ ${PROJECT_SOURCE_DIR}/.source_tag RAW_VERSION_STRING)
elseif(USE_VERSION_FROM_GIT_TAG)
	git_get_exact_tag(_GIT_TAG --tags --always)
	if (NOT ${_GIT_TAG} STREQUAL "n/a")
		set(RAW_VERSION_STRING ${_GIT_TAG})
	endif()
endif()
string(STRIP ${RAW_VERSION_STRING} RAW_VERSION_STRING)

set(VERSION_REGEX "^.?([0-9]+)\\.([0-9]+)\\.([0-9]+)-?(.*)")
string(REGEX REPLACE "${VERSION_REGEX}" "\\1" WINPR_VERSION_MAJOR "${RAW_VERSION_STRING}")
string(REGEX REPLACE "${VERSION_REGEX}" "\\2" WINPR_VERSION_MINOR "${RAW_VERSION_STRING}")
string(REGEX REPLACE "${VERSION_REGEX}" "\\3" WINPR_VERSION_REVISION "${RAW_VERSION_STRING}")
string(REGEX REPLACE "${VERSION_REGEX}" "\\4" WINPR_VERSION_SUFFIX "${RAW_VERSION_STRING}")

set(WINPR_VERSION "${WINPR_VERSION_MAJOR}.${WINPR_VERSION_MINOR}.${WINPR_VERSION_REVISION}")
set(WINPR_API_VERSION "${WINPR_VERSION_MAJOR}")
if (WINPR_VERSION_SUFFIX)
	set(WINPR_VERSION_FULL "${WINPR_VERSION}-${WINPR_VERSION_SUFFIX}")
else()
	set(WINPR_VERSION_FULL "${WINPR_VERSION}")
endif()

if(NOT IOS)
	check_include_files(stdint.h WINPR_HAVE_STDINT_H)
	check_include_files(inttypes.h WINPR_HAVE_INTTYPES_H)
else(NOT IOS)
	set(WINPR_HAVE_STDINT_H 1)
	set(WINPR_HAVE_INTTYPES_H 1)
endif(NOT IOS)

if (NOT IOS)
	check_include_files(stdbool.h WINPR_HAVE_STDBOOL_H)
	if (NOT WINPR_HAVE_STDBOOL_H)
		include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../compat/stdbool)
	endif()
else()
	set(WINPR_HAVE_STDBOOL_H 1)
endif()

set(CMAKE_THREAD_PREFER_PTHREAD TRUE)

if(NOT IOS)
	find_package(Threads REQUIRED)
endif()

# Include files
if(NOT IOS)
	check_include_files(fcntl.h HAVE_FCNTL_H)
	check_include_files(aio.h HAVE_AIO_H)
	check_include_files(sys/timerfd.h HAVE_SYS_TIMERFD_H)
	check_include_files(unistd.h HAVE_UNISTD_H)
	check_include_files(inttypes.h HAVE_INTTYPES_H)
	check_include_files(sys/filio.h HAVE_SYS_FILIO_H)
	check_include_files(sys/sockio.h HAVE_SYS_SOCKIO_H)
	check_include_files(syslog.h HAVE_SYSLOG_H)
	check_include_files(sys/select.h HAVE_SYS_SELECT_H)
	check_include_files(sys/eventfd.h HAVE_SYS_EVENTFD_H)
	if (HAVE_SYS_EVENTFD_H)
		check_symbol_exists(eventfd_read sys/eventfd.h WITH_EVENTFD_READ_WRITE)
	endif()

	include(CheckFunctionExists)
	check_function_exists(getlogin_r HAVE_GETLOGIN_R)
	check_function_exists(getpwuid_r HAVE_GETPWUID_R)
else()
	set(HAVE_FCNTL_H 1)
	set(HAVE_UNISTD_H 1)
	set(HAVE_INTTYPES_H 1)
	set(HAVE_SYS_FILIO_H 1)
endif()

if(NOT IOS)
	check_struct_has_member("struct tm" tm_gmtoff time.h HAVE_TM_GMTOFF)
else()
	set(HAVE_TM_GMTOFF 1)
endif()

if(NOT WIN32 AND NOT IOS)
	if (NOT HAVE_PTHREAD_MUTEX_TIMEDLOCK_LIB)
		CHECK_LIBRARY_EXISTS(pthreads pthread_mutex_timedlock "" HAVE_PTHREAD_MUTEX_TIMEDLOCK_LIBS)
	endif (NOT HAVE_PTHREAD_MUTEX_TIMEDLOCK_LIB)

	if (NOT HAVE_PTHREAD_MUTEX_TIMEDLOCK_SYMBOL)
		CHECK_LIBRARY_EXISTS(pthread pthread_mutex_timedlock "" HAVE_PTHREAD_MUTEX_TIMEDLOCK_LIB)
	endif (NOT HAVE_PTHREAD_MUTEX_TIMEDLOCK_SYMBOL)

	list(APPEND CMAKE_REQUIRED_LIBRARIES pthread)
	CHECK_SYMBOL_EXISTS(pthread_mutex_timedlock pthread.h HAVE_PTHREAD_MUTEX_TIMEDLOCK_SYMBOL)

	if (HAVE_PTHREAD_MUTEX_TIMEDLOCK_SYMBOL OR HAVE_PTHREAD_MUTEX_TIMEDLOCK_LIB OR HAVE_PTHREAD_MUTEX_TIMEDLOCK_LIBS)
		set(HAVE_PTHREAD_MUTEX_TIMEDLOCK ON)
	endif (HAVE_PTHREAD_MUTEX_TIMEDLOCK_SYMBOL OR HAVE_PTHREAD_MUTEX_TIMEDLOCK_LIB OR HAVE_PTHREAD_MUTEX_TIMEDLOCK_LIBS)
	list(REMOVE_ITEM CMAKE_REQUIRED_LIBRARIES pthread)
endif()

set(OPENSSL_FEATURE_TYPE "RECOMMENDED")
	set(OPENSSL_FEATURE_PURPOSE "cryptography")
	set(OPENSSL_FEATURE_DESCRIPTION "encryption, certificate validation, hashing functions")

	set(MBEDTLS_FEATURE_TYPE "OPTIONAL")
	set(MBEDTLS_FEATURE_PURPOSE "cryptography")
	set(MBEDTLS_FEATURE_DESCRIPTION "encryption, certificate validation, hashing functions")

	find_feature(OpenSSL ${OPENSSL_FEATURE_TYPE} ${OPENSSL_FEATURE_PURPOSE} ${OPENSSL_FEATURE_DESCRIPTION})
	find_feature(MbedTLS ${MBEDTLS_FEATURE_TYPE} ${MBEDTLS_FEATURE_PURPOSE} ${MBEDTLS_FEATURE_DESCRIPTION})

	if(OPENSSL_FOUND)
		add_definitions("-DWITH_OPENSSL")
	endif()

	if(MBEDTLS_FOUND)
		add_definitions("-DWITH_MBEDTLS")
	endif()

    if (NOT OPENSSL_FOUND AND NOT MBEDTLS_FOUND)
        message(FATAL_ERROR "OpenSSL or MBedTLS are required, none enabled/found")
    endif()

	# Include directories
	include_directories(BEFORE ${CMAKE_CURRENT_BINARY_DIR})

	# Configure files
	add_definitions("-DHAVE_CONFIG_H")
	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

	enable_testing()

	if(MSVC)
		set(TESTING_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
	else()
		set(TESTING_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/Testing")
endif()

if (NOT WIN32)
	set(P11_FEATURE_TYPE "OPTIONAL")
	set(P11_FEATURE_PURPOSE "PKCS11")
	set(P11_FEATURE_DESCRIPTION "encryption, certificate validation, hashing functions")

	find_feature(Pkcs11 ${P11_FEATURE_TYPE} ${P11_FEATURE_PURPOSE} ${P11_FEATURE_DESCRIPTION})
	if (PKCS11_FOUND)
		add_definitions("-DWITH_PKCS11")
	endif()
endif()

# Default to release build type
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Release")
endif()

if(BUILD_SHARED_LIBS)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DWINPR_DLL")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DWINPR_EXPORTS")

# Enable 64bit file support on linux and FreeBSD.
if("${CMAKE_SYSTEM_NAME}" MATCHES "Linux" OR FREEBSD)
	add_definitions("-D_FILE_OFFSET_BITS=64")
endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)

add_subdirectory(include)

add_subdirectory(libwinpr)

if(NOT ANDROID AND NOT IOS AND NOT UWP AND WITH_WINPR_TOOLS)
	add_subdirectory(tools)
endif()

if(BUILD_TESTING)
	add_subdirectory(test)
endif()

install_freerdp_man(wlog.7 7)
# Exporting

export(PACKAGE winpr)

SetFreeRDPCMakeInstallDir(WINPR_CMAKE_INSTALL_DIR "WinPR${WINPR_VERSION_MAJOR}")

set(WINPR_INCLUDE_DIR "include/winpr${WINPR_VERSION_MAJOR}")

configure_package_config_file(WinPRConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/WinPRConfig.cmake
	INSTALL_DESTINATION ${WINPR_CMAKE_INSTALL_DIR}
	PATH_VARS WINPR_INCLUDE_DIR)

write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/WinPRConfigVersion.cmake
	VERSION ${WINPR_VERSION} COMPATIBILITY SameMajorVersion)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/WinPRConfig.cmake ${CMAKE_CURRENT_BINARY_DIR}/WinPRConfigVersion.cmake
	DESTINATION ${WINPR_CMAKE_INSTALL_DIR})

install(EXPORT WinPRTargets DESTINATION ${WINPR_CMAKE_INSTALL_DIR})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/winpr.pc.in ${CMAKE_CURRENT_BINARY_DIR}/winpr${WINPR_VERSION_MAJOR}.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/winpr${WINPR_VERSION_MAJOR}.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/buildflags.h.in ${CMAKE_CURRENT_BINARY_DIR}/buildflags.h)
