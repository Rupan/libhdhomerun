cmake_minimum_required(VERSION 2.8.4)

IF(NOT DEFINED CMAKE_BUILD_TYPE)
   SET(CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "Choose the type of
build, options are: None(CMAKE_CXX_FLAGS or CMAKE_C_FLAGS used) Debug
Release RelWithDebInfo MinSizeRel.")
ELSE()
   SET(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build,
options are: None(CMAKE_CXX_FLAGS or CMAKE_C_FLAGS used) Debug Release
RelWithDebInfo MinSizeRel.")
ENDIF()

project(SiliconDust)

IF(APPLE)
  SET(CMAKE_OSX_ARCHITECTURES "i386;x86_64")
ELSEIF(UNIX)
  SET(CMAKE_INSTALL_PREFIX "/usr")
ENDIF()

include(CheckLibraryExists)

set(SOURCES
  hdhomerun_channels.c
  hdhomerun_channelscan.c
  hdhomerun_control.c
  hdhomerun_debug.c
  hdhomerun_device.c
  hdhomerun_discover.c
  hdhomerun_pkt.c
  hdhomerun_video.c
)

set(HEADERS
  hdhomerun.h
  hdhomerun_channels.h
  hdhomerun_channelscan.h
  hdhomerun_control.h
  hdhomerun_debug.h
  hdhomerun_device.h
  hdhomerun_device_selector.h
  hdhomerun_discover.h
  hdhomerun_pkt.h
  hdhomerun_sock.h
  hdhomerun_types.h
  hdhomerun_os.h
  hdhomerun_video.h
)

if(WIN32)
  LIST(APPEND HEADERS hdhomerun_os_windows.h)
  LIST(APPEND SOURCES hdhomerun_os_windows.c)
  LIST(APPEND SOURCES hdhomerun_sock_windows.c)
else()
  LIST(APPEND HEADERS hdhomerun_os_posix.h)
  LIST(APPEND SOURCES hdhomerun_os_posix.c)
  LIST(APPEND SOURCES hdhomerun_sock_posix.c)
endif()

check_library_exists(pthread pthread_create "" HAVE_LIBPTHREAD)
if(HAVE_LIBPTHREAD)
  list(APPEND DEPLIBS pthread)
endif()

if(WIN32)
  list(APPEND DEPLIBS ws2_32 iphlpapi)
elseif(${CMAKE_HOST_SYSTEM_NAME} STREQUAL "SunOS")
  check_library_exists(socket getaddrinfo "" HAVE_LIBSOCKET)
  if(HAVE_LIBSOCKET)
    list(APPEND DEPLIBS socket)
  endif()
elseif(${CMAKE_HOST_SYSTEM_NAME} STREQUAL "Linux")
  check_library_exists(rt clock_gettime "" HAVE_LIBRT)
  if(HAVE_LIBRT)
    list(APPEND DEPLIBS rt)
  endif()
endif()

add_library(hdhomerun-shared SHARED ${SOURCES})
if(UNIX AND NOT APPLE)
  set_property(TARGET hdhomerun-shared PROPERTY VERSION "20161117")
  set_property(TARGET hdhomerun-shared PROPERTY SOVERSION 1)
endif()
target_link_libraries(hdhomerun-shared ${DEPLIBS})

add_library(hdhomerun-static STATIC ${SOURCES})
target_link_libraries(hdhomerun-static ${DEPLIBS})

# https://cmake.org/pipermail/cmake/2007-January/012669.html
SET_TARGET_PROPERTIES(hdhomerun-static PROPERTIES OUTPUT_NAME hdhomerun CLEAN_DIRECT_OUTPUT 1)
SET_TARGET_PROPERTIES(hdhomerun-shared PROPERTIES OUTPUT_NAME hdhomerun CLEAN_DIRECT_OUTPUT 1)

if(WIN32)
  SET_TARGET_PROPERTIES(hdhomerun-static PROPERTIES DEFINE_SYMBOL "LIBHDHOMERUN_DLLEXPORT")
  SET_TARGET_PROPERTIES(hdhomerun-shared PROPERTIES DEFINE_SYMBOL "LIBHDHOMERUN_DLLEXPORT")
endif()

add_executable(hdhomerun_config hdhomerun_config.c)
target_link_libraries(hdhomerun_config hdhomerun-shared)

include_directories(${PROJECT_SOURCE_DIR})

install(FILES ${HEADERS} DESTINATION include/hdhomerun)
install(TARGETS hdhomerun-shared hdhomerun-static hdhomerun_config
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)

