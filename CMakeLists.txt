# Copyright 2018-2022, Collabora, Ltd.
# SPDX-License-Identifier: BSL-1.0

cmake_minimum_required(VERSION 3.10.2)
project(thingyTHING VERSION 0.0.1)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)


if(NOT MSVC)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pedantic -Wall -Wextra -Wno-unused-parameter -Werror=incompatible-pointer-types")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wno-unused-parameter")
	set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--no-undefined")
	set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -Wl,--no-undefined")
endif()

# Default to PIC code
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(MONADO_SRC_XRT ${CMAKE_CURRENT_SOURCE_DIR}/monado/src/xrt)

### Set configuration options for Monado
set(XRT_FEATURE_COMPOSITOR_MAIN OFF) # we are not using this
set(XRT_FEATURE_COMPOSITOR_NULL OFF) # we are not using this
set(XRT_FEATURE_OPENXR OFF) # we need to replace this to pull together a different combination of modules
set(XRT_FEATURE_IPC OFF)
set(XRT_FEATURE_OPENXR OFF)
set(XRT_FEATURE_RENDERDOC OFF)
set(XRT_FEATURE_SERVICE OFF)
set(XRT_FEATURE_SERVICE_SYSTEMD OFF)
set(XRT_FEATURE_STEAMVR_PLUGIN OFF) # for now. we'll see what transpires
set(XRT_FEATURE_TRACING OFF) #ditto
set(XRT_FEATURE_SLAM OFF) #Not using it

set(XRT_FEATURE_CLIENT_DEBUG_GUI OFF)

set(XRT_HAVE_SDL2 OFF) # Don't need this - for debug UI

set(XRT_FEATURE_STANDALONE_VIVE_CONFIG ON)
set(XRT_FEATURE_STANDALONE_MERCURY_HANDTRACKING ON)


# should be unnecessary but VS caching. delete later.
set(XRT_BUILD_DRIVER_VIVE OFF)
set(XRT_BUILD_DRIVER_HANDTRACKING OFF)
set(XRT_BUILD_DRIVER_EUROC OFF)


# Make sure we have pretty colours
option (FORCE_COLORED_OUTPUT "Always produce ANSI-colored output (GNU/Clang only)." TRUE)

add_subdirectory(monado)

message("${CMAKE_CURRENT_SOURCE_DIR}")

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")



find_package(
	OpenCV
	REQUIRED
	COMPONENTS
		core
		calib3d
		highgui
		imgproc
		imgcodecs
		features2d
		video
		CONFIG
	)

# Dear god what is this
function(ocv_add_external_target name inc link def)
  if(BUILD_SHARED_LIBS AND link)
    set(imp IMPORTED)
  endif()
  add_library(ocv.3rdparty.${name} INTERFACE ${imp})
  if(def)
    if(NOT (CMAKE_VERSION VERSION_LESS "3.11.0"))  # https://gitlab.kitware.com/cmake/cmake/-/merge_requests/1264 : eliminates "Cannot specify compile definitions for imported target" error message
      target_compile_definitions(ocv.3rdparty.${name} INTERFACE "${def}")
    else()
      set_target_properties(ocv.3rdparty.${name} PROPERTIES INTERFACE_COMPILE_DEFINITIONS "${def}")
    endif()
  endif()
  if(inc)
    if(NOT (CMAKE_VERSION VERSION_LESS "3.11.0"))  # https://gitlab.kitware.com/cmake/cmake/-/merge_requests/1264 : eliminates "Cannot specify compile definitions for imported target" error message
      target_include_directories(ocv.3rdparty.${name} SYSTEM INTERFACE "$<BUILD_INTERFACE:${inc}>")
    else()
      set_target_properties(ocv.3rdparty.${name} PROPERTIES
          INTERFACE_INCLUDE_DIRECTORIES "$<BUILD_INTERFACE:${inc}>"
          INTERFACE_SYSTEM_INCLUDE_DIRECTORIES "$<BUILD_INTERFACE:${inc}>"
      )
    endif()
  endif()
  if(link)
    # When cmake version is greater than or equal to 3.11, INTERFACE_LINK_LIBRARIES no longer applies to interface library
    # See https://github.com/opencv/opencv/pull/18658
    if(CMAKE_VERSION VERSION_LESS 3.11)
      set_target_properties(ocv.3rdparty.${name} PROPERTIES
        INTERFACE_LINK_LIBRARIES "${link}")
    else()
      target_link_libraries(ocv.3rdparty.${name} INTERFACE ${link})
    endif()
  endif()
  # to install used target only upgrade CMake
  if(NOT BUILD_SHARED_LIBS
      AND CMAKE_VERSION VERSION_LESS "3.13.0"  # https://gitlab.kitware.com/cmake/cmake/-/merge_requests/2152
  )
    install(TARGETS ocv.3rdparty.${name} EXPORT OpenCVModules)
  endif()
endfunction()

# Stolen from opencv/modules/videoio/cmake/detect_dshow.cmake
# --- VideoInput/DirectShow ---
if(NOT HAVE_DSHOW AND MSVC AND NOT MSVC_VERSION LESS 1500)
  set(HAVE_DSHOW TRUE)
endif()

if(NOT HAVE_DSHOW)
  check_include_file(dshow.h HAVE_DSHOW)
endif()

if(HAVE_DSHOW)
  ocv_add_external_target(dshow "" "" "HAVE_DSHOW")
endif()

#add_library(xrt-pthreads INTERFACE)
#if(WIN32)
#	find_package(pthreads_windows REQUIRED)
#	target_link_libraries(xrt-pthreads INTERFACE PThreads4W::PThreads4W_CXXEXC)
#else()
#	set(CMAKE_THREAD_PREFER_PTHREAD ON)
#	find_package(Threads)
#	target_link_libraries(xrt-pthreads INTERFACE Threads::Threads)
#	target_compile_definitions(xrt-pthreads INTERFACE _GNU_SOURCE)
#endif()

add_executable(
	THINGY 	src/camera_demo.cpp src/videoInput.h src/videoInput.cpp # src/cap_dshow.cpp src/cap_dshow.hpp
)

target_link_libraries(
	THINGY
	PRIVATE
	${OpenCV_LIBRARIES}
  xrt-interfaces aux_util aux_vive t_ht_mercury
)

target_include_directories(
	THINGY
	PRIVATE
  # xrt-interfaces
  # aux_vive
)



add_subdirectory(src)

#target_include_directories(
#	THINGY
#	PRIVATE
#	${OpenCV_HEADERS}
#)

