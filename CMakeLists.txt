cmake_minimum_required(VERSION 2.6)

project(minizip C)

set(INSTALL_BIN_DIR "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Installation directory for executables")
set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE PATH "Installation directory for libraries")
set(INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include/minizip" CACHE PATH "Installation directory for headers")
set(INSTALL_MAN_DIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Installation directory for manual pages")
set(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")

set(VERSION_MAJOR "1")
set(VERSION_MINOR "2")
set(VERSION_PATCH "8")
set(VERSION_STRING "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")

option(BUILD_SHARED_LIBS  "Build shared libs." ON)
option(BUILD_TESTS "Build demos." ON)
option(BUILD_DEMOS  "Build demos." ON)
option(WITH_AES  "Build AES" OFF)

if(UNIX OR MINGW)
  set(WITH_AES ON)
endif()
include (CheckIncludeFile)

check_include_file("unistd.h" HAVE_UNISTD_H)
check_include_file("sys/types.h" HAVE_SYS_TYPES_H)
check_include_file("sys/stat.h" HAVE_SYS_STAT_H)
check_include_file("stdint.h" HAVE_STDINT_H)
check_include_file("dlfcn.h" HAVE_DLFCN_H)
check_include_file("stddef.h"    HAVE_STDDEF_H)
check_include_file("ctype.h" HAVE_CTYPE_H)
check_include_file("stdlib.h" HAVE_STDLIB_H)
check_include_file("inttypes.h" HAVE_INTTYPES_H)
check_include_file("strings.h" HAVE_STRINGS_H)
check_include_file("string.h" HAVE_STRING_H)

# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

include(SuperFind)

if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
  endif()
  
if (HAVE_CTYPE_H AND HAVE_STDLIB_H)
  set(STDC_HEADERS 1)
endif ()

set(MINIZIP_SRCS
  ioapi.c
  unzip.c
  zip.c)

if(WIN32)
  list(APPEND MINIZIP_SRCS iowin32.c)
endif()

set(MINIZIP_INCS
  minizipdef.h
  crypt.h 
  ioapi.h
  zip.h
  unzip.h)

if(WIN32)
  list(APPEND MINIZIP_INCS iowin32.h)
endif()

if(WITH_AES)
  add_definitions(-DHAVE_AES)
  list(APPEND MINIZIP_SRCS
    aes/aescrypt.c
    aes/aeskey.c
    aes/aestab.c
    aes/entropy.c
    aes/fileenc.c
    aes/hmac.c
    aes/prng.c
    aes/pwd2key.c
    aes/sha1.c)
  include_directories(${CMAKE_SOURCE_DIR}/aes)
endif()

set(LIB_NAME minizip)

add_library(${LIB_NAME} ${MINIZIP_SRCS})

super_find_package(ZLIB)
#The above macro calls find_package(ZLIB) and if not found build it download and build it for you!
#find_package(ZLIB)

include_directories(${ZLIB_INCLUDE_DIR})
add_dependencies_if_needed(${LIB_NAME} ZLIB)

target_link_libraries(${LIB_NAME} ${ZLIB_LIBRARY})
set_target_properties(${LIB_NAME} PROPERTIES SOVERSION ${VERSION_STRING})
set_target_properties(${LIB_NAME} PROPERTIES DEFINE_SYMBOL MINIZIP_DLL)

if(BUILD_DEMOS)
  add_executable(miniunz miniunz.c)
  target_link_libraries(miniunz ${LIB_NAME} ${ZLIB_LIBRARY})
  
  install(TARGETS miniunz 
    RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
    ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
    LIBRARY DESTINATION "${INSTALL_LIB_DIR}" )
  
  add_executable(minizipz minizip.c)
  target_link_libraries(minizipz ${LIB_NAME} ${ZLIB_LIBRARY})
  
  install(TARGETS minizipz
    RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
    ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
    LIBRARY DESTINATION "${INSTALL_LIB_DIR}" )
endif()

if(NOT WIN32)
configure_file( ${CMAKE_SOURCE_DIR}/cmake/minizip.pc.cmakein
  ${CMAKE_BINARY_DIR}/minizip.pc @ONLY)

install(FILES ${CMAKE_BINARY_DIR}/minizip.pc
  DESTINATION
  "${INSTALL_PKGCONFIG_DIR}")

endif()
set(MINIZIP_VERSION ${VERSION_STRING})

if(WIN32 AND NOT CYGWIN)
  set(DEF_INSTALL_CMAKE_DIR cmake)
else()
  set(DEF_INSTALL_CMAKE_DIR lib/cmake/minizip)
endif()

set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH
  "Installation directory for CMake files")
 
# Make relative paths absolute (needed later on)
foreach(p LIB BIN INCLUDE CMAKE)
  set(var INSTALL_${p}_DIR)
  if(NOT IS_ABSOLUTE "${${var}}")
    set(${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
  endif()
endforeach()

install(TARGETS ${LIB_NAME}
  EXPORT MiniZipTargets
  RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}" )

install(FILES ${MINIZIP_INCS} DESTINATION "${INSTALL_INC_DIR}")

if(BUILD_TESTS)
  enable_testing()
endif()

set(CONFIG_TYPE "NOCONFIG")
if(NOT "${CMAKE_BUILD_TYPE}" STREQUAL "")
  string(TOUPPER ${CMAKE_BUILD_TYPE} CONFIG_TYPE)
endif()

export(TARGETS ${LIB_NAME}
  FILE "${CMAKE_BINARY_DIR}/MiniZipTargets.cmake")

# Create the ZLIBConfig.cmake and ZLIBConfigVersion files
file(RELATIVE_PATH REL_INCLUDE_DIR "${INSTALL_CMAKE_DIR}"
  "${INSTALL_INC_DIR}")

# ... for the build tree
#set(CONF_INCLUDE_DIRS "${CMAKE_SOURCE_DIR}/include")
set(CONF_INCLUDE_DIRS "${CMAKE_INSTALL_PREFIX}/include")

configure_file(cmake/MiniZipConfig.cmake.in
  "${CMAKE_BINARY_DIR}/MiniZipConfig.cmake" @ONLY)

# ... for the install tree
set(CONF_INCLUDE_DIRS "\${INSTALL_CMAKE_DIR}/${REL_INCLUDE_DIR}")
configure_file(cmake/MiniZipConfig.cmake.in
  "${CMAKE_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/MiniZipConfig.cmake" @ONLY)
# ... for both
configure_file(cmake/MiniZipConfigVersion.cmake.in
  "${CMAKE_BINARY_DIR}/MiniZipConfigVersion.cmake" @ONLY)
 
install(FILES
  "${CMAKE_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/MiniZipConfig.cmake"
  "${CMAKE_BINARY_DIR}/MiniZipConfigVersion.cmake"
  DESTINATION ${DEF_INSTALL_CMAKE_DIR})
 
# Install the export set for use with the install-tree
install(EXPORT MiniZipTargets DESTINATION ${DEF_INSTALL_CMAKE_DIR})