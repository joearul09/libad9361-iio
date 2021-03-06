# Copyright (C) 2015 Analog Devices, Inc.
#
# This library is free software; you can redistribute it and/or
# modify it under the terms of the GNU Lesser General Public
# License as published by the Free Software Foundation; either
# version 2.1 of the License, or (at your option) any later version.
#
# This library is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
# Lesser General Public License for more details.

cmake_minimum_required(VERSION 2.8.7)
project(ad9361 C)

set(LIBAD9361_VERSION_MAJOR 0)
set(LIBAD9361_VERSION_MINOR 1)
set(LIBAD9361_VERSION ${LIBAD9361_VERSION_MAJOR}.${LIBAD9361_VERSION_MINOR})

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	option(OSX_PACKAGE "Create a OSX package" ON)
	set(CMAKE_MACOSX_RPATH ON)
	set(SKIP_INSTALL_ALL ${OSX_PACKAGE})
endif()

set(BUILD_SHARED_LIBS ON CACHE BOOL "Build shared libraries")

if (NOT WIN32)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
endif()

add_definitions(-DLIBAD9361_EXPORTS=1)

set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib"
	CACHE PATH "Installation directory for libraries")
mark_as_advanced(INSTALL_LIB_DIR)

find_library(LIBIIO_LIBRARIES iio)
find_path(LIBIIO_INCLUDEDIR iio.h)

set(LIBAD9361_HEADERS ad9361.h)

include_directories(${CMAKE_CURRENT_SOURCE_DIR} ${LIBIIO_INCLUDEDIR})
add_library(ad9361 ad9361_multichip_sync.c ad9361_baseband_auto_rate.c ${LIBAD9361_HEADERS})
set_target_properties(ad9361 PROPERTIES VERSION ${LIBAD9361_VERSION} SOVERSION ${LIBAD9361_VERSION_MAJOR} FRAMEWORK TRUE PUBLIC_HEADER "${LIBAD9361_HEADERS}")
target_link_libraries(ad9361 LINK_PRIVATE ${LIBIIO_LIBRARIES})

if (MSVC)
	set_target_properties(ad9361 PROPERTIES OUTPUT_NAME libad9361)
endif()

set(LIBAD9361_PC ${CMAKE_CURRENT_BINARY_DIR}/libad9361.pc)
configure_file(libad9361.pc.cmakein ${LIBAD9361_PC} @ONLY)
install(FILES ${LIBAD9361_PC} DESTINATION ${INSTALL_LIB_DIR}/pkgconfig)

if(NOT SKIP_INSTALL_ALL)
	install(TARGETS ad9361
		ARCHIVE DESTINATION lib
		LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
		RUNTIME DESTINATION bin
		FRAMEWORK DESTINATION lib
		PUBLIC_HEADER DESTINATION include)
endif()

# Create an installer if compiling for OSX
if(OSX_PACKAGE)
	set(LIBAD9361_PKG ${CMAKE_CURRENT_BINARY_DIR}/libad9361-${LIBAD9361_VERSION}.pkg)
	set(LIBAD9361_TEMP_PKG ${CMAKE_CURRENT_BINARY_DIR}/libad6361-${LIBAD9361_VERSION}-temp.pkg)
	set(LIBAD9361_DISTRIBUTION_XML ${CMAKE_CURRENT_BINARY_DIR}/Distribution.xml)
	set(LIBAD9361_FRAMEWORK_DIR ${CMAKE_CURRENT_BINARY_DIR}/ad9361.framework)
	configure_file(Distribution.xml.cmakein ${LIBAD9361_DISTRIBUTION_XML} @ONLY)

	find_program(PKGBUILD_EXECUTABLE
		NAMES pkgbuild
		DOC "OSX Package builder (pkgbuild)")
	mark_as_advanced(PKGBUILD_EXECUTABLE)

	find_program(PRODUCTBUILD_EXECUTABLE
		NAMES productbuild
		DOC "OSX Package builder (productbuild)")
	mark_as_advanced(PRODUCTBUILD_EXECUTABLE)

	add_custom_command(OUTPUT ${LIBAD9361_PKG}
		COMMAND ${PKGBUILD_EXECUTABLE}
			--component ${LIBAD9361_FRAMEWORK_DIR}
			--identifier com.adi.ad9361 --version ${LIBAD9361_VERSION}
			--install-location /Library/Frameworks ${LIBAD9361_TEMP_PKG}
		COMMAND ${PRODUCTBUILD_EXECUTABLE}
			--distribution ${LIBAD9361_DISTRIBUTION_XML} ${LIBAD9361_PKG}
		COMMAND ${CMAKE_COMMAND} -E remove ${LIBAD9361_TEMP_PKG}
		DEPENDS ad9361 ${LIBAD9361_DISTRIBUTION_XML}
	)

	if (PKGBUILD_EXECUTABLE AND PRODUCTBUILD_EXECUTABLE)
		add_custom_target(libad9361-pkg ALL DEPENDS ${LIBAD9361_PKG})

		install(CODE "execute_process(COMMAND /usr/sbin/installer -pkg ${LIBAD9361_PKG} -target /)")
	else()
		message(WARNING "Missing pkgbuild or productbuild: OSX installer won't be created.")
	endif()
endif()
