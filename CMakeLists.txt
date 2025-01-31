# CMake definitions for AIS-catcher

cmake_minimum_required(VERSION 3.0)
project(AIS-catcher)

# Common compiler flags and options.

option(MSVC_VCPKG "For MSVC use VCPKG libraries." OFF)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
	set(OPTIMIZATION_FLAGS "-Ofast")
	add_compile_options("-Wno-psabi")
elseif(CMAKE_CXX_COMPILER_ID MATCHES "CLANG")
	set(OPTIMIZATION_FLAGS "-Ofast")
elseif(MSVC)
	set(OPTIMIZATION_FLAGS "-DWIN32_LEAN_AND_MEAN /EHsc /Oi /Ot /fp:fast")
endif()

set(CMAKE_CXX_FLAGS "${OPTIMIZATION_FLAGS}")

# Use pkg-config
include(FindPkgConfig)
find_package(Threads)

# for MSVC we link to the PothosSDR install
if(MSVC AND NOT MSVC_VCPKG)
	SET(POTHOSSDR_BINARY_DIR $ENV{PROGRAMFILES}\\PothosSDR\\bin)
	SET(POTHOSSDR_LIBRARY_DIR $ENV{PROGRAMFILES}\\PothosSDR\\lib)
	SET(POTHOSSDR_INCLUDE_DIR $ENV{PROGRAMFILES}\\PothosSDR\\include)
endif()

# Define the libraries for inclusion
if(MSVC)
	set(ADDITIONAL_LIBRARIES ws2_32.lib)
endif()

# Find libusb, pthread
if(NOT MSVC)
	pkg_check_modules(PKG_LIBUSB libusb-1.0)
	find_path(LIBUSB_INCLUDE_DIR libusb.h HINT ${PKG_LIBUSB_INCLUDE_DIRS} PATH_SUFFIXES libusb-1.0)
	find_library(LIBUSB_LIBRARY usb-1.0 HINT ${PKG_LIBUSB_LIBRARY_DIRS})
elseif(NOT MSVC_VCPKG)
	find_file(LIBUSB_DLL libusb-1.0.dll HINTS ${POTHOSSDR_BINARY_DIR})
	find_file(PTHREAD_DLL pthreadVC2.dll HINTS ${POTHOSSDR_BINARY_DIR})
else()
	set(LIBUSB_INCLUDE_DIR "")
	set(LIBUSB_LIBRARY "")
endif()

# Find RTL-SDR library.

if(NOT MSVC)

	pkg_check_modules(PKG_RTLSDR librtlsdr)
	find_path(RTLSDR_INCLUDE_DIR rtl-sdr.h HINT ${PKG_RTLSDR_INCLUDE_DIRS})
	find_library(RTLSDR_LIBRARY rtlsdr HINT ${PKG_RTLSDR_LIBRARY_DIRS})

	if(RTLSDR_INCLUDE_DIR AND RTLSDR_LIBRARY)

		message(STATUS "RTLSDR: found - ${RTLSDR_INCLUDE_DIR}, ${RTLSDR_LIBRARY}")
		add_definitions(-DHASRTLSDR)

		set(RTLSDR_INCLUDE_DIRS ${RTLSDR_INCLUDE_DIR} ${LIBUSB_INCLUDE_DIR})
		set(RTLSDR_LIBRARIES ${RTLSDR_LIBRARY} ${LIBUSB_LIBRARY})

		if(PKG_RTLSDR_VERSION LESS "0.6")
			Message(STATUS "RTLSDR: no support for biastee")
			add_definitions(-DLIBRTLSDR_LEGACY)
		endif()

	else()
		Message(STATUS "RTLSDR: not found.")
	endif()

elseif(MSVC_VCPKG)

	find_package(rtlsdr)
	if(rtlsdr_FOUND)
		add_definitions(-DHASRTLSDR)
		set(RTLSDR_LIBRARIES rtlsdr::rtlsdr)
		Message(STATUS "RTLSDR: found (VCPKG) - " ${RTLSDR_LIBRARIES})
	else()
		Message(STATUS "RTLSDR: not found (VCPKG).")
	endif()

else()

	find_path(RTLSDR_INCLUDE_DIR rtl-sdr.h HINTS ${POTHOSSDR_INCLUDE_DIR})
	find_library(RTLSDR_LIBRARY rtlsdr.lib HINTS ${POTHOSSDR_LIBRARY_DIR})
	find_file(RTLSDR_DLL rtlsdr.dll HINTS ${POTHOSSDR_BINARY_DIR})

	if(RTLSDR_INCLUDE_DIR AND RTLSDR_LIBRARY AND RTLSDR_DLL)
		Message(STATUS "RTLSDR: found (PothosSDR) - " ${RTLSDR_INCLUDE_DIR}, ${RTLSDR_LIBRARY}, ${RTLSDR_DLL})
		set(COPY_RTLSDR_DLL TRUE)
		set(COPY_PTHREAD_LIBUSB_DLL TRUE)
		add_definitions(-DHASRTLSDR)
		set(RTLSDR_INCLUDE_DIRS ${RTLSDR_INCLUDE_DIR})
		set(RTLSDR_LIBRARIES ${RTLSDR_LIBRARY})
	else()
		Message(STATUS "RTLSDR: not found (PothosSDR).")
	endif()
endif()


# Find SDRplay library.
if(NOT MSVC)

	find_path(SDRPLAY_INCLUDE_DIR sdrplay_api.h)
	find_library(SDRPLAY_LIBRARY sdrplay_api)

	if(SDRPLAY_INCLUDE_DIR AND SDRPLAY_LIBRARY)
		message(STATUS "SDRPLAY: found - ${SDRPLAY_INCLUDE_DIR}, ${SDRPLAY_LIBRARY}")
		add_definitions(-DHASSDRPLAY)
		set(SDRPLAY_INCLUDE_DIRS ${SDRPLAY_INCLUDE_DIR})
		set(SDRPLAY_LIBRARIES ${SDRPLAY_LIBRARY})
	else()
		message(STATUS "SDRPLAY: not found.")
	endif()

else()

	set(SDRPLAY_API_DIR $ENV{PROGRAMFILES}\\SDRplay\\API)
	if(CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(SDRPLAY_API_LIB_DIR ${SDRPLAY_API_DIR}\\x64\\)
	elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
		set(SDRPLAY_API_LIB_DIR ${SDRPLAY_API_DIR}\\x32\\)
	endif()

	find_path(SDRPLAY_INCLUDE_DIR sdrplay_api.h HINTS ${SDRPLAY_API_DIR}\\inc)
	find_library(SDRPLAY_LIBRARY sdrplay_api.lib HINTS ${SDRPLAY_API_LIB_DIR})
	find_file(SDRPLAY_DLL sdrplay_api.dll HINTS ${SDRPLAY_API_LIB_DIR})

	if(SDRPLAY_INCLUDE_DIR AND SDRPLAY_LIBRARY AND SDRPLAY_DLL)
		message(STATUS "SDRPLAY: found - ${SDRPLAY_INCLUDE_DIR}, ${SDRPLAY_LIBRARY}, ${SDRPLAY_DLL}")
		add_definitions(-DHASSDRPLAY)
		set(COPY_SDRPLAY_DLL TRUE)
		set(SDRPLAY_INCLUDE_DIRS ${SDRPLAY_INCLUDE_DIR})
		set(SDRPLAY_LIBRARIES ${SDRPLAY_LIBRARY})
	else()
		message(STATUS "SDRPLAY: not found. Assumed directory ${SDRPLAY_API_LIB_DIR}")
	endif()

endif()

# Find Airspy library.

if(NOT MSVC)

	pkg_check_modules(PKG_AIRSPY libairspy)
	find_path(AIRSPY_INCLUDE_DIR airspy.h HINT ${PKG_AIRSPY_INCLUDE_DIRS})
	find_library(AIRSPY_LIBRARY airspy HINT ${PKG_AIRSPY_LIBRARY_DIRS})

	if(AIRSPY_INCLUDE_DIR AND AIRSPY_LIBRARY)

		message(STATUS "AIRSPY: found ${AIRSPY_INCLUDE_DIR}, ${AIRSPY_LIBRARY}")
		add_definitions(-DHASAIRSPY)
		set(AIRSPY_INCLUDE_DIRS ${AIRSPY_INCLUDE_DIR} ${LIBUSB_INCLUDE_DIR})
		set(AIRSPY_LIBRARIES ${AIRSPY_LIBRARY} ${LIBUSB_LIBRARY})

	else()
		Message(STATUS "AIRSPY: not found.")
	endif()

elseif(MSVC_VCPKG)

	Message("AIRSPY: not found (VCPKG)")

else()

	find_path(AIRSPY_INCLUDE_DIR airspy.h HINTS ${POTHOSSDR_INCLUDE_DIR}\\libairspy)
	find_library(AIRSPY_LIBRARY airspy.lib HINTS ${POTHOSSDR_BINARY_DIR})
	find_file(AIRSPY_DLL airspy.dll HINTS ${POTHOSSDR_BINARY_DIR})

	if(AIRSPY_INCLUDE_DIR AND AIRSPY_LIBRARY AND AIRSPY_DLL)
		Message(STATUS "AIRSPYHF: found (PothosSDR) - " ${AIRSPY_INCLUDE_DIR}, ${AIRSPY_LIBRARY}, ${AIRSPY_DLL})
		set(COPY_AIRSPY_DLL TRUE)
		set(COPY_PTHREAD_LIBUSB_DLL TRUE)
		add_definitions(-DHASAIRSPY)
		set(AIRSPY_INCLUDE_DIRS ${AIRSPY_INCLUDE_DIR})
		set(AIRSPY_LIBRARIES ${AIRSPY_LIBRARY})
	else()
		Message(STATUS "AIRSPY: not found.")
	endif()
endif()

# Find Airspy library.

if(NOT MSVC)

	pkg_check_modules(PKG_AIRSPYHF libairspyhf)
	find_path(AIRSPYHF_INCLUDE_DIR airspyhf.h HINT ${PKG_AIRSPYHF_INCLUDE_DIRS})
	find_library(AIRSPYHF_LIBRARY airspyhf HINT ${PKG_AIRSPYHF_LIBRARY_DIRS})

	if(AIRSPYHF_INCLUDE_DIR AND AIRSPYHF_LIBRARY)

		message(STATUS "AIRSPYHF: found - ${AIRSPYHF_INCLUDE_DIR}, ${AIRSPYHF_LIBRARY}")
		add_definitions(-DHASAIRSPYHF)
		set(AIRSPYHF_INCLUDE_DIRS ${AIRSPYHF_INCLUDE_DIR} ${LIBUSB_INCLUDE_DIR})
		set(AIRSPYHF_LIBRARIES ${AIRSPYHF_LIBRARY} ${LIBUSB_LIBRARY})

	else()
		Message(STATUS "AIRSPYHF: not found.")
	endif()

elseif(MSVC_VCPKG)

	Message("AIRSPYHF: not found (VCPKG)")

else()

	find_path(AIRSPYHF_INCLUDE_DIR AIRSPYHF.h HINTS ${POTHOSSDR_INCLUDE_DIR}\\libairspyhf)
	find_library(AIRSPYHF_LIBRARY AIRSPYHF.lib HINTS ${POTHOSSDR_BINARY_DIR})
	find_file(AIRSPYHF_DLL AIRSPYHF.dll HINTS ${POTHOSSDR_BINARY_DIR})

	if(AIRSPYHF_INCLUDE_DIR AND AIRSPYHF_LIBRARY AND AIRSPYHF_DLL)
		Message(STATUS "AIRSPYHF: found (PothosSDR) - " ${AIRSPYHF_INCLUDE_DIR}, ${AIRSPYHF_LIBRARY}, ${AIRSPYHF_DLL})
		set(COPY_AIRSPYHF_DLL TRUE)
		set(COPY_PTHREAD_LIBUSB_DLL TRUE)
		add_definitions(-DHASAIRSPYHF)
		set(AIRSPYHF_INCLUDE_DIRS ${AIRSPYHF_INCLUDE_DIR})
		set(AIRSPYHF_LIBRARIES ${AIRSPYHF_LIBRARY})
	else()
		Message(STATUS "AIRSPYHF: not found (PothosSDR).")
	endif()

endif()

# Find HACKRF library.

if(NOT MSVC)

	pkg_check_modules(PKG_HACKRF libhackrf)
	find_path(HACKRF_INCLUDE_DIR hackrf.h HINT ${PKG_HACKRF_INCLUDE_DIRS} /usr/include/libhackrf)
	find_library(HACKRF_LIBRARY hackrf HINT ${PKG_HACKRF_LIBRARY_DIRS})

	if(HACKRF_INCLUDE_DIR AND HACKRF_LIBRARY)

		message(STATUS "HACKRF: found - ${HACKRF_INCLUDE_DIR}, ${HACKRF_LIBRARY}")
		add_definitions(-DHASHACKRF)
		set(HACKRF_INCLUDE_DIRS ${HACKRF_INCLUDE_DIR} ${LIBUSB_INCLUDE_DIR})
		set(HACKRF_LIBRARIES ${HACKRF_LIBRARY} ${LIBUSB_LIBRARY})

	else()
		Message(STATUS "HACKRF: not found.")
	endif()

elseif(MSVC_VCPKG)

	Message("HACKRF: not found (VCPKG)")

else()

	find_path(HACKRF_INCLUDE_DIR hackrf.h HINTS ${POTHOSSDR_INCLUDE_DIR}\\libhackrf)
	find_library(HACKRF_LIBRARY hackrf.lib HINTS ${POTHOSSDR_LIBRARY_DIR})
	find_file(HACKRF_DLL hackrf.dll HINTS ${POTHOSSDR_BINARY_DIR})

	if(HACKRF_INCLUDE_DIR AND HACKRF_LIBRARY AND HACKRF_DLL)
		Message(STATUS "HACKRF: found (PothosSDR) - " ${HACKRF_INCLUDE_DIR}, ${HACKRF_LIBRARY}, ${HACKRF_DLL})
		set(COPY_HACKRF_DLL TRUE)
		set(COPY_PTHREAD_LIBUSB_DLL TRUE)
		add_definitions(-DHASHACKRF)
		set(HACKRF_INCLUDE_DIRS ${HACKRF_INCLUDE_DIR})
		set(HACKRF_LIBRARIES ${HACKRF_LIBRARY})
	else()
		Message(STATUS "HACKRF: not found (PothosSDR)")
	endif()

endif()

# Find libzmq
if(NOT MSVC)
	pkg_check_modules(PKG_LIBZMQ libzmq)
	find_path(LIBZMQ_INCLUDE_DIR zmq.h HINT ${PKG_LIBZMQ_INCLUDE_DIRS})
	find_library(LIBZMQ_LIBRARY zmq HINT ${PKG_LIBZMQ_LIBRARY_DIRS})

	if(LIBZMQ_INCLUDE_DIR AND LIBZMQ_LIBRARY)
		message(STATUS "ZMQ: found - ${LIBZMQ_INCLUDE_DIR}, ${LIBZMQ_LIBRARY}")
		add_definitions(-DHASZMQ)
		set(ZMQ_INCLUDE_DIRS ${LIBZMQ_INCLUDE_DIR})
		set(ZMQ_LIBRARIES ${LIBZMQ_LIBRARY})
	else()
		message(STATUS "ZMQ: not found.")
	endif()

elseif(MSVC_VCPKG)

	find_package(ZeroMQ)

	if(ZeroMQ_FOUND)
		add_definitions(-DHASZMQ)
		set(ZMQ_LIBRARIES libzmq)
		Message(STATUS "ZMQ: found (VCPKG) - " ${ZMQ_LIBRARIES})
	else()
		Message(STATUS "ZMQ: not found (VCPKG).")
	endif()

else()
	find_path(ZMQ_INCLUDE_DIR zmq.h HINTS ${POTHOSSDR_INCLUDE_DIR})
	find_library(ZMQ_LIBRARY libzmq-v142-mt-4_1_8.lib HINTS ${POTHOSSDR_LIBRARY_DIR})
	find_file(ZMQ_DLL libzmq-v142-mt-4_1_8.dll HINTS ${POTHOSSDR_BINARY_DIR})

	if(ZMQ_INCLUDE_DIR AND ZMQ_LIBRARY AND ZMQ_DLL)
		Message(STATUS "ZMQ: found (PothosSDR) - " ${ZMQ_INCLUDE_DIR}, ${ZMQ_LIBRARY}, ${ZMQ_DLL})
		set(COPY_ZMQ_DLL TRUE)
		set(COPY_PTHREAD_LIBUSB_DLL TRUE)
		add_definitions(-DHASZMQ)
		set(ZMQ_INCLUDE_DIRS ${ZMQ_INCLUDE_DIR})
		set(ZMQ_LIBRARIES ${ZMQ_LIBRARY})
	else()
		Message(STATUS "ZMQ: not found (PothosSDR)")
	endif()
endif()

set(CPP
	Application/Main.cpp
	Device/AIRSPYHF.cpp Device/FileWAV.cpp Device/RTLSDR.cpp Device/SDRPLAY.cpp DSP/Demod.cpp DSP/Model.cpp Library/AIS.cpp
	Device/FileRAW.cpp Device/HACKRF.cpp Device/RTLTCP.cpp Device/ZMQ.cpp
	Library/Utilities.cpp
	Device/AIRSPY.cpp
	DSP/DSP.cpp
	IO/IO.cpp)

set(HEADER
	Application/AIS-catcher.h
	Device/Device.h Device/FileWAV.h Device/RTLTCP.h DSP/Demod.h DSP/Filters.h Library/AIS.h Library/Signals.h
	Device/AIRSPY.h Device/FIFO.h Device/HACKRF.h Device/SDRPLAY.h DSP/DSP.h DSP/Model.h Library/Common.h Library/Stream.h
	Device/AIRSPYHF.h Device/FileRAW.h Device/RTLSDR.h Device/ZMQ.h DSP/FFT.h IO/IO.h Library/Utilities.h)

set(APP_INCLUDES . ./Library ./DSP ./Application ./IO)

# Executable

add_executable(AIS-catcher ${CPP} ${HEADER})

include_directories(
	${AIRSPYHF_INCLUDE_DIRS} ${AIRSPY_INCLUDE_DIRS} ${HACKRF_INCLUDE_DIRS} ${RTLSDR_INCLUDE_DIRS} ${ZMQ_INCLUDE_DIRS} ${SDRPLAY_INCLUDE_DIRS}
	${APP_INCLUDES})

target_link_libraries(AIS-catcher
	${AIRSPY_LIBRARIES} ${AIRSPYHF_LIBRARIES} ${RTLSDR_LIBRARIES} ${HACKRF_LIBRARIES} ${ZMQ_LIBRARIES} ${SDRPLAY_LIBRARIES}
	${ADDITIONAL_LIBRARIES} Threads::Threads)

# Copying DLLs to final location if needed
if(COPY_SDRPLAY_DLL)
	add_custom_command(TARGET AIS-catcher POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${SDRPLAY_DLL} ${CMAKE_CURRENT_BINARY_DIR})
endif()

if(COPY_RTLSDR_DLL)
	add_custom_command(TARGET AIS-catcher POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${RTLSDR_DLL} ${CMAKE_CURRENT_BINARY_DIR})
endif()

if(COPY_AIRSPY_DLL)
	add_custom_command(TARGET AIS-catcher POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${AIRSPY_DLL} ${CMAKE_CURRENT_BINARY_DIR})
endif()

if(COPY_AIRSPYHF_DLL)
	add_custom_command(TARGET AIS-catcher POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${AIRSPYHF_DLL} ${CMAKE_CURRENT_BINARY_DIR})
endif()

if(COPY_HACKRF_DLL)
	add_custom_command(TARGET AIS-catcher POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${HACKRF_DLL} ${CMAKE_CURRENT_BINARY_DIR})
endif()

if(COPY_ZMQ_DLL)
	add_custom_command(TARGET AIS-catcher POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${ZMQ_DLL} ${CMAKE_CURRENT_BINARY_DIR})
endif()

if(COPY_PTHREAD_LIBUSB_DLL)
	add_custom_command(TARGET AIS-catcher POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${LIBUSB_DLL} ${CMAKE_CURRENT_BINARY_DIR})
	add_custom_command(TARGET AIS-catcher POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${PTHREAD_DLL} ${CMAKE_CURRENT_BINARY_DIR})
endif()

# Installation
install(TARGETS AIS-catcher DESTINATION bin)

# End of CMakeLists.txt
