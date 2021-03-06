cmake_minimum_required(VERSION 2.8.5)
project(libbladeRF C)

################################################################################
# Version information
################################################################################

set(VERSION_INFO_MAJOR  0)
set(VERSION_INFO_MINOR  10)
set(VERSION_INFO_PATCH  0)
if(NOT DEFINED VERSION_INFO_EXTRA)
    set(VERSION_INFO_EXTRA "git")
endif()
include(Version)

set(VERSION "${VERSION_INFO}")
set(LIBVER "${VERSION_INFO_BASE}")

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/src/version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/src/version.h
    @ONLY
)

include(GNUInstallDirs)

# Fall back to just "lib" if the item provided by GNUInstallDirs doesn't exist
# For example, on Ubuntu 13.10 with CMake 2.8.11.2,
# /usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} doesn't exist.
if (NOT EXISTS "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}")
    message(STATUS "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR} does not exist. Defaulting libbladeRF install location to ${CMAKE_INSTALL_PREFIX}/lib.")
    set(CMAKE_INSTALL_LIBDIR lib)
endif()

################################################################################
# Configuration options
################################################################################

option(ENABLE_LIBBLADERF_LOGGING "Enable log messages in libbladeRF" ON)

##############################
# Backend Support
##############################
option(ENABLE_BACKEND_LIBUSB "Enable libusb backend support"
    ON
)

option(LIBUSB_DETACH_KERNEL_DRIVER
    "Enable calls to libusb_detach_kernel_driver(). Not supported in Windows."
    ON
)

option(ENABLE_BACKEND_LINUX_DRIVER
    "Enable Linux driver backend support."
    OFF
)

# Ensure we've got at least one backend enabled
if(NOT ENABLE_BACKEND_LIBUSB AND NOT ENABLE_BACKEND_LINUX_DRIVER)
    message(FATAL_ERROR
            "No libbladeRF backends are enabled. "
            "Please enable one or more backends." )
endif()

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/src/backend_config.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/src/backend_config.h
    @ONLY
)

################################################################################
# Additional compiler options
################################################################################
if(ENABLE_LIBBLADERF_LOGGING)
    add_definitions(-DLOGGING_ENABLED)
endif()


if("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU" OR
    "${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" )

    # Doing this manually instead of via add_compiler_export_flags() since
    # the GenerateExportHeader module appears to break for C-only projects:
    #
    # http://www.cmake.org/pipermail/cmake-commits/2012-August/013142.html
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
endif()

################################################################################
# Include paths
################################################################################
set(LIBBLADERF_INCLUDES
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${BLADERF_HOST_COMMON_INCLUDE_DIRS}
    ${BLADERF_FW_COMMON_INCLUDE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}/src
)

################################################################################
# Build dependencies
################################################################################
if(ENABLE_BACKEND_LIBUSB)
    find_package(LibUSB)
    if(NOT LIBUSB_FOUND)
        message(FATAL_ERROR "libusb-1.0 not found. This is required to use the libbladeRF libusb backend. For binary releases, try setting LIBUSB_PATH.")
    else(NOT LIBUSB_FOUND)
        if(LIBUSB_HAVE_GET_VERSION)
            add_definitions(-DHAVE_LIBUSB_GET_VERSION)
        endif(LIBUSB_HAVE_GET_VERSION)

        set(LIBBLADERF_INCLUDES ${LIBBLADERF_INCLUDES} ${LIBUSB_INCLUDE_DIRS})
    endif(NOT LIBUSB_FOUND)
endif(ENABLE_BACKEND_LIBUSB)

include_directories(${LIBBLADERF_INCLUDES})

################################################################################
# Configure source files
################################################################################
set(LIBBLADERF_SOURCE
        src/backend.c
        src/bladerf.c
        src/bladerf_priv.c
        src/file_ops.c
        src/lms.c
        src/si5338.c
        src/version.h
        src/device_identifier.c
        src/devinfo.c
        src/flash.c
        src/image.c
        ${BLADERF_HOST_COMMON_SOURCE_DIR}/sha256.c
        ${BLADERF_HOST_COMMON_SOURCE_DIR}/conversions.c
        ${BLADERF_HOST_COMMON_SOURCE_DIR}/log.c
)

if(LIBUSB_FOUND AND ENABLE_BACKEND_LIBUSB)
    set(LIBBLADERF_SOURCE ${LIBBLADERF_SOURCE} src/backend/libusb.c)
endif()

if(ENABLE_BACKEND_LINUX_DRIVER)
    set(LIBBLADERF_SOURCE ${LIBBLADERF_SOURCE} src/backend/linux.c)
endif()

add_library(libbladerf_shared SHARED ${LIBBLADERF_SOURCE})

################################################################################
# Build configuration
################################################################################

if(ENABLE_BACKEND_LIBUSB)
    target_link_libraries(libbladerf_shared ${LIBUSB_LIBRARIES})
endif()

# Adjust our output name
set_target_properties(libbladerf_shared PROPERTIES OUTPUT_NAME bladeRF)

# Set shared library version
set_target_properties(libbladerf_shared PROPERTIES SOVERSION ${VERSION_INFO_MAJOR})


################################################################################
# Generate pkg-config file
################################################################################
add_subdirectory(include)
foreach(inc ${LIBBLADERF_INCLUDE_DIR})
    list(APPEND LIBBLADERF_PC_CFLAGS "-I${inc}")
endforeach()

foreach(lib ${LIBBLADERF_LIBRARY_DIRS})
    list(APPEND LIBBLADERF_PC_PRIV_LIBS "-L${lib}")
endforeach()

set(LIBBLADERF_PC_PREFIX ${CMAKE_INSTALL_PREFIX})
set(LIBBLADERF_PC_EXEC_PREFIX \${prefix})
set(LIBBLADERF_PC_LIBDIR \${exec_prefix}/${CMAKE_INSTALL_LIBDIR})
set(LIBBLADERF_PC_INCLUDEDIR \${prefix}/include)
set(LIBBLADERF_PC_VERSION ${VERSION})

# Use space-delimiter in the .pc file, rather than CMake's semicolon separator
string(REPLACE ";" " " LIBBLADERF_PC_CFLAGS "${LIBBLADERF_PC_CFLAGS}")
string(REPLACE ";" " " LIBBLADERF_PC_LIBS "${LIBBLADERF_PC_LIBS}")

# Unset these to avoid hard-coded paths in a cross-environment
if(CMAKE_CROSSCOMPILING)
    unset(LIBBLADERF_PC_CFLAGS)
    unset(LIBBLADERF_PC_LIBS)
endif()

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/libbladeRF.pc.in
    ${CMAKE_CURRENT_BINARY_DIR}/libbladeRF.pc
    @ONLY
)

install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/libbladeRF.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig/
)

################################################################################
# Library installation information
################################################################################
install(TARGETS libbladerf_shared
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} # .so/.dylib
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} # .a/.lib
    RUNTIME DESTINATION ${CMAKE_INSTALL_LIBDIR} # .dll
)


################################################################################
# Informational output
################################################################################
message(STATUS "libbladeRF version: ${VERSION_INFO}")

################################################################################
# Library tests
################################################################################
add_subdirectory(tests)

################################################################################
# Doxygen documentation
################################################################################
if(BUILD_DOCUMENTATION)
    find_package(Doxygen)
    if(DOXYGEN_FOUND)
        message(STATUS "Configured to build libbladeRF API documentation.")
        configure_file(
            ${CMAKE_CURRENT_SOURCE_DIR}/doc/doxygen/Doxyfile.in
            ${CMAKE_CURRENT_BINARY_DIR}/doc/doxygen/Doxyfile
            @ONLY
            )

        add_custom_target(doc
            ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/doc/doxygen/Doxyfile
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc/doxygen
            COMMENT "Generating libbladeRF API documentation via Doxygen in: ${CMAKE_CURRENT_BINARY_DIR}/doc/doxygen"
        )
    else(DOXYGEN_FOUND)
        message(WARNING "Could not find Doxygen. libbladeRF API documentation will not be built.")
    endif(DOXYGEN_FOUND)
endif(BUILD_DOCUMENTATION)
