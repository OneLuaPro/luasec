# ------------------------------------------------------------------------------
# CMake for luasec for OneLuaPro
# ------------------------------------------------------------------------------
# Setup with Visual Studio 17 2022 generator for x64
# ------------------------------------------------------------------------------
# Visual Studio is a mutlti-configuration generator
# https://stackoverflow.com/questions/24460486/
#
# Basic instructions for out-of-source build
# ------------------------------------------
# mkdir build && cd build
# cmake .. -G "Visual Studio 17 2022" -A x64
# cmake --build . --config Release
# cmake --install . --config Release
# 
# Available architectures (-A ...) are: Win32, x64, ARM, ARM64

# ------------------------------------------------------------------------------
# General definitions
cmake_minimum_required(VERSION 3.31 FATAL_ERROR)

# ------------------------------------------------------------------------------
# find liblua installation and version info
if(NOT LUA_HINTS)
  if(WIN32)
    set(LUA_HINTS "c:/Apps")
  endif()
endif()
find_package(liblua REQUIRED CONFIG HINTS ${LUA_HINTS})
if(liblua_FOUND)
  message(STATUS "liblua version        : ${liblua_VERSION}")
  message(STATUS "liblua install prefix : ${LIBLUA_INSTALLDIR}")
  message(STATUS "liblua include dir    : ${LIBLUA_INCLUDEDIR}")
  message(STATUS "liblua lib dir        : ${LIBLUA_LIBDIR}")
else()
  message(FATAL_ERROR "Unable to find liblua version ${liblua_VERSION}.")
endif()
# Note: liblua_VERSION is set by find_package() directly. LIBLUA_INSTALLDIR,
# LIBLUA_INCLUDEDIR and LIBLUA_LIBDIR are set by libluaConfigVersion.cmake in
# <prefix>/share/cmake/liblua.

# ------------------------------------------------------------------------------
# Installation prefix directory - automatically set from find_package()
# Needs to be defined before project definition statement - for whatever reason
if(NOT CMAKE_INSTALL_PREFIX)
  set(CMAKE_INSTALL_PREFIX ${LIBLUA_INSTALLDIR})
endif()

# ------------------------------------------------------------------------------
# Project defintion
project(luasec LANGUAGES C)

# ------------------------------------------------------------------------------
# Other settings
set(CMAKE_VERBOSE_MAKEFILE ON)
if(NOT CMAKE_BUILD_TYPE)
  # Visual Studio generators irgore this setting
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()
# create valiable with upper-case content out of CMAKE_BUILD_TYPE
string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UC)

# ------------------------------------------------------------------------------
# Setup GNU-alike installatin directories
include (GNUInstallDirs)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})
set(INSTALL_LIBDIR
  ${CMAKE_INSTALL_LIBDIR} CACHE PATH "Installation directory for libraries")
set(INSTALL_BINDIR
  ${CMAKE_INSTALL_BINDIR} CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDEDIR
  ${CMAKE_INSTALL_INCLUDEDIR} CACHE PATH "Installation directory for header files")
set(INSTALL_DOCDIR
  ${CMAKE_INSTALL_DOCDIR} CACHE PATH "Installation directory for documentation")
set(INSTALL_MANDIR
  ${CMAKE_INSTALL_MANDIR} CACHE PATH "Installation directory for manpages")
set(INSTALL_DATAROOTDIR
  ${CMAKE_INSTALL_DATAROOTDIR} CACHE PATH "Installation directory for data")
# Lua-specific installation dirs
set(INSTALL_TOP_CDIR
  ${INSTALL_LIBDIR}/lua/${liblua_VERSION_MAJOR}.${liblua_VERSION_MINOR})
set(INSTALL_TOP_LDIR
  ${INSTALL_DATAROOTDIR}/lua/${liblua_VERSION_MAJOR}.${liblua_VERSION_MINOR})

# ------------------------------------------------------------------------------
# luasec DLL
add_library(luasec SHARED)
# setup lua include directory
target_include_directories(luasec PRIVATE ${LIBLUA_INCLUDEDIR})
# setup platform-specific sources, compile and linker options
if(WIN32 AND NOT MinGW)
  # Important definitions for DLL and symbol export
  add_definitions(-DLUA_BUILD_AS_DLL -DLUA_LIB)
  # set deviating library output name
  set_target_properties(luasec PROPERTIES PREFIX "")
  set_target_properties(luasec PROPERTIES OUTPUT_NAME ssl)
  # Compile options
  target_compile_definitions(luasec PRIVATE WIN32_LEAN_AND_MEAN
    WAITFD_R=1 WAITFD_W=2 WAITFD_E=4 WAITFD_C=3
    _WINDLL _WIN32 _CRT_SECURE_NO_WARNINGS
    WITH_LUASOCKET OPENSSL_API_COMPAT=0x10101000L)
  # We silently assume LibreSSL is available with headers and static lib
  target_link_options(luasec PRIVATE /LIBPATH:${LIBLUA_LIBDIR} liblua.lib
    ssl.lib crypto.lib libluasocket.lib ws2_32.lib bcrypt.lib)
else()
  message(FATAL_ERROR "Not yet fully implemented.")
endif()
# plattform-independend sources
target_sources(luasec PRIVATE
  src/compat.h src/context.c src/ec.c src/options.h src/ssl.c src/x509.h
  src/config.c src/context.h src/ec.h src/options.c src/ssl.h src/x509.c
)
# Install
install(TARGETS luasec RUNTIME DESTINATION ${INSTALL_TOP_CDIR})
# install lua files to defined locations
install(FILES src/ssl.lua DESTINATION ${INSTALL_TOP_LDIR})
install(FILES src/https.lua DESTINATION ${INSTALL_TOP_LDIR}/ssl)

# ------------------------------------------------------------------------------
# Install docs and samples
install(DIRECTORY doc/ DESTINATION ${INSTALL_DOCDIR})	# "/" omits leading dir
install(DIRECTORY samples DESTINATION ${INSTALL_DOCDIR})
