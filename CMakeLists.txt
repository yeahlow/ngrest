cmake_minimum_required (VERSION 2.8.4)
project (ngrest)

set(NGREST_VERSION "0.2.0")

include(CheckCXXCompilerFlag)
include(CheckLibraryExists)
include(CheckIncludeFileCXX)

check_cxx_compiler_flag(-std=gnu++11 HAS_CXX11)
if (HAS_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11")
endif()

if (NOT APPLE)
    check_cxx_compiler_flag(-Wl,--no-undefined HAS_NO_UNDEFINED)
    if (HAS_NO_UNDEFINED)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--no-undefined")
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--no-undefined")
        set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--no-undefined")
    endif()
endif()

check_cxx_compiler_flag(-pedantic HAS_PEDANTIC)
if (HAS_PEDANTIC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic")
endif()

check_cxx_compiler_flag(-Wall HAS_WALL)
if (HAS_WALL)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
endif()

check_library_exists(dl dlopen "" HAS_DL)
if (NOT APPLE)
    check_include_file_cxx(sys/epoll.h HAS_EPOLL)
    if (HAS_EPOLL)
        add_definitions(-DHAS_EPOLL)
    endif()
endif()

if (MINGW)
    add_definitions(-D_WIN32_WINNT=0x0501)
    set(CMAKE_SHARED_LIBRARY_PREFIX "")
    set(CMAKE_SHARED_MODULE_PREFIX "")
    set(CMAKE_SHARED_LIBRARY_SUFFIX ".dll")
    set(CMAKE_SHARED_MODULE_SUFFIX ".dll")
endif()

check_library_exists(pthread pthread_create "" HAS_PTHREAD)
if ("$ENV{WITH_THREAD_LOCK}" STREQUAL "1")
    add_definitions(-DNGREST_THREAD_LOCK)
    set(LINK_PTHREAD "${HAS_PTHREAD}")
endif()

if (CMAKE_BUILD_TYPE MATCHES "DEBUG")
    add_definitions(-DDEBUG)
    # link pthread in debug/single thread version
    # to detect incorrect posting to event loop from non-main thread
    set(LINK_PTHREAD "${HAS_PTHREAD}")
endif()

set(PROJECT_DEPLOY_DIR ${PROJECT_BINARY_DIR}/deploy)
set(PROJECT_INCLUDE_DIR ${PROJECT_DEPLOY_DIR}/include)
set(PROJECT_SHARE_DIR ${PROJECT_DEPLOY_DIR}/share/ngrest)
set(PROJECT_SERVICES_DIR ${PROJECT_SHARE_DIR}/services)
set(PROJECT_FILTERS_DIR ${PROJECT_SHARE_DIR}/filters)
set(PROJECT_CODEGEN_DIR ${PROJECT_SHARE_DIR}/codegen)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_DEPLOY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_DEPLOY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_DEPLOY_DIR}/bin)
set(TESTS_OUTPUT_DIRECTORY ${PROJECT_DEPLOY_DIR}/tests)
set(EXAMPLES_OUTPUT_DIRECTORY ${PROJECT_DEPLOY_DIR}/examples)

FUNCTION(PREPEND var prefix)
   SET(listVar "")
   FOREACH(f ${ARGN})
      LIST(APPEND listVar "${prefix}/${f}")
   ENDFOREACH(f)
   SET(${var} "${listVar}" PARENT_SCOPE)
ENDFUNCTION(PREPEND)

FUNCTION(CODEGEN_FILES outVar prefix)
   SET(listVar "${prefix}/ServiceGroupImpl.h" "${prefix}/ServiceGroupImpl.cpp")
   FOREACH(file ${ARGN})
      get_filename_component(fileBase "${file}" NAME_WE)
      LIST(APPEND listVar "${prefix}/${fileBase}Wrapper.h")
      LIST(APPEND listVar "${prefix}/${fileBase}Wrapper.cpp")
   ENDFOREACH(file)
   SET(${outVar} "${listVar}" PARENT_SCOPE)
ENDFUNCTION(CODEGEN_FILES)

include_directories("${PROJECT_INCLUDE_DIR}")

add_subdirectory(core)
add_subdirectory(tools)
add_subdirectory(services)
add_subdirectory(modules)

if ("$ENV{WITH_TESTS}" STREQUAL "1")
    add_subdirectory(tests)
endif()

if ("$ENV{WITH_EXAMPLES}" STREQUAL "1")
    add_subdirectory(examples)
endif()
