﻿cmake_minimum_required(VERSION 3.0)

# For checks in subdirectories
set(InMVSDK TRUE)

# Custom CMake Modules
list(INSERT CMAKE_MODULE_PATH 0 "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Project & Versioning
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(MVSDK "JK2MV SDK")
set(MVSDKURL "https://jk2mv.org")
if(NOT InMV)
	set(ProjectName ${MVSDK} CACHE STRING "Project Name")
	project(${ProjectName})
endif()

include(GetGitRevisionDescription)
git_describe(MVSDK_VERSION --tags)
if(NOT MVSDK_VERSION OR MVSDK_VERSION STREQUAL "GIT-NOTFOUND" OR MVSDK_VERSION STREQUAL "HEAD-HASH-NOTFOUND")
	set(MVSDK_VERSION "UNKNOWN")
	message(WARNING "Could not find git or git repository. Install git and make sure you actually cloned the repository (Directory named .git must exist) for correct versioning.")
endif()

message(STATUS "${MVSDK} Version: ${MVSDK_VERSION}")

# Languages
enable_language(C)

if(CMAKE_CONFIGURATION_TYPES)
	set(MultiConfigGenerator TRUE)
endif()

# Customizable options
if(NOT InMV)
	option(BuildQVMs "Whether to create targets for building QVM files" ON)
else()
	set(BuildQVMs ON)
endif()

# Architecture
if(NOT InMV)
	if(WIN32)
		set(ARCH_X86 ON)
		set(GlobalDefines ${GlobalDefines} "WIN32")
		if(CMAKE_SIZEOF_VOID_P MATCHES "8")
			set(GlobalDefines ${GlobalDefines} "ARCH_X86_64" "WIN64")
			set(Architecture "x64")
			set(ARCH_X86_64 ON)
		else()
			set(GlobalDefines ${GlobalDefines} "ARCH_X86")
			set(Architecture "x86")
			set(ARCH_X86_32 ON)
		endif()
	elseif(APPLE)
		set(ARCH_X86 ON)
		set(GlobalDefines ${GlobalDefines} "MACOS_X")
		if(CMAKE_SYSTEM_PROCESSOR MATCHES "^i.86$")
			set(GlobalDefines ${GlobalDefines} "ARCH_X86")
			set(Architecture "i386")
			set(ARCH_X86_32 ON)
		elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^x86.64$")
			set(GlobalDefines ${GlobalDefines} "ARCH_X86_64")
			set(Architecture "x86_64")
			set(ARCH_X86_64 ON)
		endif()
	else()
		if(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm")
			set(GlobalDefines ${GlobalDefines} "ARCH_ARM32")
			set(Architecture "arm")
			set(ARCH_ARM32 ON)
		elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^i.86$")
			set(GlobalDefines ${GlobalDefines} "ARCH_X86")
			set(Architecture "i386")
			set(ARCH_X86 ON)
			set(ARCH_X86_32 ON)
		elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^x86.64$")
			set(GlobalDefines ${GlobalDefines} "ARCH_X86_64")
			set(Architecture "amd64")
			set(ARCH_X86 ON)
			set(ARCH_X86_64 ON)
		else()
			set(GlobalDefines ${GlobalDefines} "NO_VM_COMPILED")
			set(Architecture "${CMAKE_SYSTEM_PROCESSOR}")
			set(ARCH_UNKNOWN ON)
		endif()
	endif()

	message(STATUS "Architecture is ${Architecture}")
	if(ARCH_UNKNOWN)
		message(WARNING "Building for unsupported Architecture ${Architecture}.")
	endif()
endif()

# Binary names
set(MVGAME "jk2mpgame_${Architecture}")
set(MVCGAME "cgame_${Architecture}")
set(MVUI "ui_${Architecture}")

set(LCC "q3lcc")
set(CPP "q3cpp")
set(RCC "q3rcc")
set(ASM "q3asm")

# Directories
set(SRCDir "${CMAKE_CURRENT_SOURCE_DIR}/code")
set(QVMDir "${CMAKE_BINARY_DIR}/out/vm")

if(MultiConfigGenerator)
	# Visual Studio, Xcode etc. support selecting build type in the IDE and automatically append
	# the configuration type to the path
	set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/out)
	set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/out)
	set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/out)
else()
	if(NOT CMAKE_BUILD_TYPE AND NOT InMV)
		message(STATUS "No build type selected, default to Release")
		set(CMAKE_BUILD_TYPE "Release")
	endif()

	set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/out/${CMAKE_BUILD_TYPE})
	set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/out/${CMAKE_BUILD_TYPE})
	set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/out/${CMAKE_BUILD_TYPE})
endif()

# Compiler defines / settings
set(GlobalDefines ${GlobalDefines} "MISSIONPACK")
set(GlobalDefines ${GlobalDefines} "GAME_EXPORTS")

if (NOT InMV)
	set(GlobalDefines ${GlobalDefines} "_JK2")

	set(DebugDefines ${DebugDefines} "DEBUG")
	set(ReleaseDefines "FINAL_BUILD" "NDEBUG")
endif()

if (NOT InMV)
	if(MSVC)
		# Warnings
		# 4244 = conversion' conversion from 'type1' to 'type2', possible loss of data
		# 4996 = 'function': was declared deprecated
		# 4267 = 'var' : conversion from 'size_t' to 'type', possible loss of data
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4244 /wd4996 /wd4267")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4244 /wd4996 /wd4267")
	
		# Multithreaded
		set(CMAKE_CXX_FLAGS_RELEASE "/MT" CACHE TYPE INTERNAL FORCE)
		set(CMAKE_C_FLAGS_RELEASE "/MT" CACHE TYPE INTERNAL FORCE)

		# Platform
		set(GlobalDefines ${GlobalDefines} "UNICODE" "_UNICODE")

		# Optimization
		set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MP") # multiprocessor compilation
		set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MP")
		set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /O2 /Oi /Ot /Ob2 /fp:fast") # maximize speed
		set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /O2 /Oi /Ot /Ob2 /fp:fast")

		if(ARCH_X86_32)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /arch:SSE2")
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /arch:SSE2")

			set(CMAKE_ASM_MASM_FLAGS "${CMAKE_ASM_MASM_FLAGS} /safeseh") # safe exception handler
		endif()
	elseif ((${CMAKE_C_COMPILER_ID} STREQUAL GNU) OR (${CMAKE_C_COMPILER_ID} MATCHES Clang|AppleClang))
		# Additional flags for debug configuration
		set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -ggdb3")
		set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -ggdb3")

		# we need sse optimizations in debug configuration, too because of the inline assembler which is sse code
		if(ARCH_X86)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse2")
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse2")
		endif()

		# Optimization
		set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3 -ffast-math")
		set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -ffast-math")

		# General
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden")

		# Warnings
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wsign-compare -Wno-unknown-pragmas")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-unknown-pragmas -Wno-write-strings")

		if(${CMAKE_C_COMPILER_ID} STREQUAL GNU)
			if(ARCH_X86)
				set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -mfpmath=sse")
			endif()

			# Link only libs actually needed
			set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -Wl,--as-needed")
		endif()

		if(${CMAKE_CXX_COMPILER_ID} STREQUAL GNU)
			if(ARCH_X86)
				set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -mfpmath=sse")
			endif()

			# Link only libs actually needed
			# workaround for older gcc versions and c++11 std::thread: explicitly link lpthread
			# https://bugs.launchpad.net/ubuntu/+source/gcc-defaults/+bug/1228201
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wl,--no-as-needed -pthread -lpthread -Wl,--as-needed")

			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility-inlines-hidden")

			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wformat=2 -Wno-unused-but-set-variable")
		elseif("${CMAKE_CXX_COMPILER_ID}" MATCHES Clang|AppleClang)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-char-subscripts -Wno-self-assign")
		endif()
	else()
		message(FATAL_ERROR "Unsupported compiler")
	endif()
endif()

if(BuildQVMs)
	add_subdirectory("tools")
endif()

add_subdirectory(${SRCDir})
