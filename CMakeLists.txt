########################################################################
#
# Fobos SDR (agile) project
# V.T.
# LGPL
#
########################################################################
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.

########################################################################
# Project setup
########################################################################
cmake_minimum_required(VERSION 3.7.2)

project(fobos LANGUAGES C)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_EXTENSIONS OFF)

include(GNUInstallDirs)
message(>>> "CMAKE_INSTALL_LIBDIR: " ${CMAKE_INSTALL_LIBDIR})
########################################################################

########################################################################
# Select the release build type by default to get optimization flags
########################################################################
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Release")
   message(STATUS "Build type not specified: defaulting to release.")
endif(NOT CMAKE_BUILD_TYPE)
########################################################################

########################################################################
# Compiler specific setup
########################################################################
if(MSVC)
    add_definitions(
        -D_CRT_SECURE_NO_WARNINGS
        -D_TIMESPEC_DEFINED
    )
else()
    add_compile_options(
        -Wall
        -Wextra
        -Wno-unused-parameter
        -Wno-unused
        -Wsign-compare
    )
endif()
########################################################################

########################################################################
# Find build dependencies
########################################################################
find_package(PkgConfig)
if(NOT DEFINED LIBUSB_INCLUDE_DIRS OR NOT DEFINED LIBUSB_LIBRARIES)
    find_package(PkgConfig)
    if(PKG_CONFIG_FOUND)
        pkg_check_modules(LIBUSB libusb-1.0 IMPORTED_TARGET)
        if(NOT LIBUSB_FOUND)
            message(FATAL_ERROR "LibUSB 1.0 required")
        else()
            set(LIBUSB_LIBRARIES "${LIBUSB_LINK_LIBRARIES}")
        endif()
    else()
        if(MSVC)
            if(MSVC_VERSION EQUAL 1800)
                set(LIBUSB_SUFFIX VS2013)
            elseif(MSVC_VERSION EQUAL 1900)
                set(LIBUSB_SUFFIX VS2015)
            elseif((MSVC_VERSION GREATER_EQUAL 1910) AND (MSVC_VERSION LESS_EQUAL 1919))
                set(LIBUSB_SUFFIX VS2017)
            elseif((MSVC_VERSION GREATER_EQUAL 1920) AND (MSVC_VERSION LESS_EQUAL 1929))
                set(LIBUSB_SUFFIX VS2019)
            else()
                set(LIBUSB_SUFFIX VS2022)
            endif()

            if(CMAKE_SIZEOF_VOID_P EQUAL 4)
                set(LIBUSB_SUFFIX "${LIBUSB_SUFFIX}/MS32")
            else()
                set(LIBUSB_SUFFIX "${LIBUSB_SUFFIX}/MS64")
            endif()

            set(LIBUSB_SUFFIX "${LIBUSB_SUFFIX}/dll")
        elseif(MINGW)
            if(CMAKE_SIZEOF_VOID_P EQUAL 4)
                set(LIBUSB_SUFFIX "MinGW32")
            else()
                set(LIBUSB_SUFFIX "MinGW64")
            endif()

            set(LIBUSB_SUFFIX "${LIBUSB_SUFFIX}/dll/libusb-1.0.dll")
        endif()

        set(LIBUSB_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/libusb/include)
        set(LIBUSB_LIBRARIES ${CMAKE_SOURCE_DIR}/libusb/${LIBUSB_SUFFIX})
    endif()
endif()
########################################################################

########################################################################
# Install udev rules
########################################################################
if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    install(
        FILES fobos-sdr.rules
        DESTINATION "/etc/udev/rules.d"
        COMPONENT "udev"
    )
endif()
########################################################################

########################################################################
# Print Summary
########################################################################
message(>>> "Build: " ${CMAKE_BUILD_TYPE})
message(>>> "OS: " ${CMAKE_SYSTEM_NAME})
message(>>> "VS Platform: " ${CMAKE_VS_PLATFORM_NAME})
message(>>> "LIBUSB_LIBRARIES: " ${LIBUSB_LIBRARIES})
message(>>> "LIBUSB_INCLUDE_DIRS: " ${LIBUSB_INCLUDE_DIRS})
message(>>> "PROJECT_BINARY_DIR: " ${PROJECT_BINARY_DIR})

########################################################################
# Fobos SDR (agile) library
########################################################################
add_library(libfobos_sdr SHARED fobos/fobos_sdr.c)

if(MSVC)
    target_compile_options(libfobos_sdr PUBLIC "/MT")
    add_custom_command(TARGET libfobos_sdr
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/Debug
        COMMAND ${CMAKE_COMMAND} -E copy ${LIBUSB_LIBRARIES}/libusb-1.0.dll ${PROJECT_BINARY_DIR}/Debug
    )
    add_custom_command(TARGET libfobos_sdr
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/Release
        COMMAND ${CMAKE_COMMAND} -E copy ${LIBUSB_LIBRARIES}/libusb-1.0.dll ${PROJECT_BINARY_DIR}/Release
)
elseif(MINGW)
    add_custom_command(TARGET libfobos_sdr
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${LIBUSB_LIBRARIES} ${PROJECT_BINARY_DIR}/msys-usb-1.0.dll
    )
endif()

if(MSVC)
    target_link_directories(libfobos_sdr PRIVATE ${LIBUSB_LIBRARIES})
else()
    target_link_libraries(libfobos_sdr PRIVATE ${LIBUSB_LIBRARIES})
endif()

target_include_directories(libfobos_sdr PUBLIC
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}>/fobos
    ${LIBUSB_INCLUDE_DIRS}
)

set_target_properties(libfobos_sdr PROPERTIES DEFINE_SYMBOL "FOBOS_SDR_EXPORTS")
set_target_properties(libfobos_sdr PROPERTIES OUTPUT_NAME fobos_sdr)
########################################################################

########################################################################
# Install public header files
########################################################################
install(
    FILES fobos/fobos_sdr.h
    DESTINATION include
)
########################################################################

########################################################################
# Install built library files & utilities
########################################################################
install(
    TARGETS libfobos_sdr
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
########################################################################

########################################################################
# Create Pkg Config File
########################################################################
foreach(inc ${LIBUSB_INCLUDEDIR})
    list(APPEND LIBFOBOS_SDR_PC_CFLAGS "-I${inc}")
endforeach(inc)

foreach(lib ${LIBUSB_LIBRARY_DIRS})
    list(APPEND LIBFOBOS_SDR_PC_LIBS "-L${lib}")
endforeach(lib)

# use space-separation format for the pc file
string(REPLACE ";" " " LIBFOBOS_SDR_PC_CFLAGS "${LIBFOBOS_SDR_PC_CFLAGS}")
string(REPLACE ";" " " LIBFOBOS_SDR_PC_LIBS "${LIBFOBOS_SDR_PC_LIBS}")

set(prefix "${CMAKE_INSTALL_PREFIX}")
set(exec_prefix \${prefix})
set(includedir \${prefix}/include)
set(libdir \${exec_prefix}/lib)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/libfobos_sdr.pc.in
    ${CMAKE_CURRENT_BINARY_DIR}/libfobos_sdr.pc
    @ONLY
)

install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/libfobos_sdr.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)
########################################################################

########################################################################
# Create uninstall target
########################################################################
configure_file(
    ${CMAKE_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
@ONLY)

add_custom_target(uninstall
    ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
)
########################################################################

########################################################################
# Create executable fobos_sdr_devinfo
########################################################################
add_executable(fobos_sdr_devinfo
    eval/fobos_sdr_devinfo_main.c)

target_link_libraries(fobos_sdr_devinfo PRIVATE libfobos_sdr)

target_include_directories(fobos_sdr_devinfo PRIVATE ${CMAKE_SOURCE_DIR}/fobos)
########################################################################

########################################################################
# Create executable fobos_sdr_recorder
########################################################################
add_executable(fobos_sdr_recorder 
    eval/fobos_sdr_recorder_main.c 
    wav/wav_file.c)

target_link_libraries(fobos_sdr_recorder PRIVATE libfobos_sdr)

target_include_directories(fobos_sdr_recorder PRIVATE ${CMAKE_SOURCE_DIR})

target_include_directories(fobos_sdr_recorder PRIVATE ${CMAKE_SOURCE_DIR}/fobos)
########################################################################

########################################################################
# Create executable fobos_sdr_scanner
########################################################################
add_executable(fobos_sdr_scanner 
    eval/fobos_sdr_scanner_main.c 
    wav/wav_file.c)

target_link_libraries(fobos_sdr_scanner PRIVATE libfobos_sdr)

target_include_directories(fobos_sdr_scanner PRIVATE ${CMAKE_SOURCE_DIR})

target_include_directories(fobos_sdr_scanner PRIVATE ${CMAKE_SOURCE_DIR}/fobos)
########################################################################

########################################################################
# Create executable fobos_sdr_fwloader
########################################################################
add_executable(fobos_sdr_fwloader 
    eval/fobos_sdr_fwloader_main.c 
    wav/wav_file.c)

target_link_libraries(fobos_sdr_fwloader PRIVATE libfobos_sdr)

target_include_directories(fobos_sdr_fwloader PRIVATE ${CMAKE_SOURCE_DIR})

target_include_directories(fobos_sdr_fwloader PRIVATE ${CMAKE_SOURCE_DIR}/fobos)
########################################################################

########################################################################
# Directories
########################################################################
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
########################################################################
