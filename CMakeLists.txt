cmake_minimum_required(VERSION 2.8.11)
include(${CMAKE_ROOT}/Modules/ExternalProject.cmake)
SET(CMAKE_CXX_STANDARD 11)
SET(CMAKE_CONFIGURATION_TYPES ${CMAKE_BUILD_TYPE} CACHE STRING "RelWithDebInfo" FORCE)
project(Art1_Qbox)
file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/wrappers)

cmake_host_system_information(RESULT BUILD_HOST QUERY HOSTNAME)
## If we are on titan, set FFTW paths
if ("${BUILD_HOST}" MATCHES "(titan-ext)[1-7]")
	set(FFTW_DIR /opt/cray/fftw/3.3.4.11/interlagos)
	set(CUDA_TOOLKIT_ROOT_DIR "$ENV{CRAY_CUDATOOLKIT_DIR}")
	set(LIBCUDA_DIR "/opt/cray/nvidia/default/lib64")
endif("${BUILD_HOST}" MATCHES "(titan-ext)[1-7]")

if (NOT DEFINED FFTW_DIR)
	ExternalProject_Add(
	  FFTW_INSTALL
	  EXCLUDE_FROM_ALL 1
	  URL "ftp://ftp.fftw.org/pub/fftw/fftw-3.3.6-pl2.tar.gz"
	  URL_MD5 "927e481edbb32575397eb3d62535a856"
	  SOURCE_DIR "${PROJECT_BINARY_DIR}/3rdparty/fftw3"
	  CONFIGURE_COMMAND ${PROJECT_BINARY_DIR}/3rdparty/fftw3/configure --prefix=${CMAKE_INSTALL_PREFIX} --enable-shared CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER}
	  BUILD_COMMAND $(MAKE) -j 4
	)
	set(FFTW_DIR ${CMAKE_INSTALL_PREFIX})
else(NOT DEFINED FFTW_DIR)
	add_custom_target(FFTW_INSTALL)
endif(NOT DEFINED FFTW_DIR)

find_package(CUDA REQUIRED)

if (NOT DEFINED LIBCUDA_DIR)
	message(ERROR " Set -DLIBCUDA_DIR to the path containing libcuda.so")
endif(NOT DEFINED LIBCUDA_DIR)

add_subdirectory(${CMAKE_SOURCE_DIR}/src/fftw_intercept)

configure_file (
  "${PROJECT_SOURCE_DIR}/src/wrappers/qbox_configure.sh.in"
  "${PROJECT_BINARY_DIR}/wrappers/qbox_configure.sh"
  )

configure_file (
  "${PROJECT_SOURCE_DIR}/src/wrappers/cray-xt4.fixed.mk.in"
  "${PROJECT_BINARY_DIR}/wrappers/qbox.cray-xt4.fixed.mk"
  @ONLY
)
configure_file (
  "${PROJECT_SOURCE_DIR}/src/wrappers/cray-xt4.mk.in"
  "${PROJECT_BINARY_DIR}/wrappers/qbox.cray-xt4.mk"
  @ONLY
)

ExternalProject_Add(
  XERCES
  EXCLUDE_FROM_ALL 1
  URL "http://pages.cs.wisc.edu/~welton/xerces-c-3.1.4.tar.bz2"
  URL_MD5 "d04ae9d8b2dee2157c6db95fa908abfd"
  SOURCE_DIR "${PROJECT_BINARY_DIR}/3rdparty/xerces"
  CONFIGURE_COMMAND ${PROJECT_BINARY_DIR}/3rdparty/xerces/configure --prefix=${CMAKE_INSTALL_PREFIX}
  BUILD_COMMAND $(MAKE) -j 4
)

ExternalProject_Add(
  QBOX
  DEPENDS XERCES FFTW_INSTALL
  URL "http://qboxcode.org/download/qbox-1.63.5.tgz"
  URL_MD5 "da3161ab6a455793f2133dd03c04077c"
  SOURCE_DIR "${PROJECT_BINARY_DIR}/3rdparty/qbox"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND cp ${PROJECT_BINARY_DIR}/wrappers/qbox.cray-xt4.mk ${PROJECT_BINARY_DIR}/3rdparty/qbox/src/cray-xt4.mk && cd ${PROJECT_BINARY_DIR}/3rdparty/qbox/src && TARGET=cray-xt4 $(MAKE)
  INSTALL_COMMAND cp ${PROJECT_BINARY_DIR}/3rdparty/qbox/src/qb ${CMAKE_INSTALL_PREFIX}/bin/qbox
  UPDATE_COMMAND ""
)

ExternalProject_Add(
  QBOX_FIXED
  DEPENDS QBOX FFTW_INSTALL fftw_intercept
  URL "http://qboxcode.org/download/qbox-1.63.5.tgz"
  URL_MD5 "da3161ab6a455793f2133dd03c04077c"
  SOURCE_DIR "${PROJECT_BINARY_DIR}/3rdparty/qbox_fixed"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND cp ${PROJECT_BINARY_DIR}/wrappers/qbox.cray-xt4.fixed.mk ${PROJECT_BINARY_DIR}/3rdparty/qbox_fixed/src/cray-xt4.mk && cp ${PROJECT_SOURCE_DIR}/src/FourierTransform.C ${PROJECT_BINARY_DIR}/3rdparty/qbox_fixed/src/FourierTransform.C && cd ${PROJECT_BINARY_DIR}/3rdparty/qbox_fixed/src && TARGET=cray-xt4 $(MAKE)
  INSTALL_COMMAND cp ${PROJECT_BINARY_DIR}/3rdparty/qbox_fixed/src/qb ${CMAKE_INSTALL_PREFIX}/bin/qbox_fixed
  UPDATE_COMMAND ""
)
