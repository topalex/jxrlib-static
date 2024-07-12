# Copyright Mathieu Malaterre <malat@debian.org>
# BSD (Same as jxrlib)
# Based on https://jxrlib.codeplex.com/discussions/440294
cmake_minimum_required(VERSION 2.8)
project(jxrlib C)

# Need shared libs for ABI
set(BUILD_SHARED_LIBS ON)

# helper macro to preserve original Makefile convention
macro(JXR_MAKE_OBJ SET_NAME)
  foreach(src ${SRC_${SET_NAME}})
    list(APPEND OBJ_${SET_NAME} ${DIR_${SET_NAME}}/${src})
  endforeach()
endmacro()

if(NOT MSVC)
  add_definitions(-D__ANSI__)
  add_compile_options(-Wno-error=implicit-function-declaration)
endif()

include(TestBigEndian)
test_big_endian(ISBIGENDIAN)
if(ISBIGENDIAN)
  set(DEF_ENDIAN _BIG__ENDIAN_)
endif()

set(DIR_SYS image/sys)
set(DIR_DEC image/decode)
set(DIR_ENC image/encode)

set(DIR_GLUE jxrgluelib)
set(DIR_TEST jxrtestlib)
set(DIR_EXEC jxrencoderdecoder)

set(VERSION "@VERSION@")
set(SOVERSION "0")

if(NOT JXRLIB_INSTALL_BINDIR)
  set(JXRLIB_INSTALL_BINDIR "bin")
endif()

if(NOT JXRLIB_INSTALL_LIBDIR)
  set(JXRLIB_INSTALL_LIBDIR "lib")
endif()

if(NOT JXRLIB_INSTALL_INCLUDEDIR)
  set(JXRLIB_INSTALL_INCLUDEDIR "include/jxrlib")
endif()

include_directories(
  common/include
  ${DIR_SYS}
  ${DIR_GLUE}
  ${DIR_TEST}
)

# JPEG-XR
set(SRC_SYS adapthuff.c image.c strcodec.c strPredQuant.c strTransform.c perfTimerANSI.c)
JXR_MAKE_OBJ(SYS)
set(SRC_DEC decode.c postprocess.c segdec.c strdec.c strInvTransform.c strPredQuantDec.c JXRTranscode.c)
JXR_MAKE_OBJ(DEC)
set(SRC_ENC encode.c segenc.c strenc.c strFwdTransform.c strPredQuantEnc.c)
JXR_MAKE_OBJ(ENC)

add_library(jpegxr STATIC ${OBJ_ENC} ${OBJ_DEC} ${OBJ_SYS})
set_property(TARGET jpegxr
  PROPERTY COMPILE_DEFINITIONS __ANSI__ DISABLE_PERF_MEASUREMENT ${DEF_ENDIAN}
)
set_property(TARGET jpegxr PROPERTY LINK_INTERFACE_LIBRARIES "")
set_property(TARGET jpegxr PROPERTY COMPILE_FLAGS -w)
set_property(TARGET jpegxr PROPERTY VERSION ${VERSION})
set_property(TARGET jpegxr PROPERTY SOVERSION ${SOVERSION})
install(TARGETS jpegxr
  EXPORT JXRLibTargets
  RUNTIME DESTINATION ${JXRLIB_INSTALL_BINDIR} COMPONENT Applications
  LIBRARY DESTINATION ${JXRLIB_INSTALL_LIBDIR} COMPONENT Libraries
)

# JXR-GLUE
set(SRC_GLUE JXRGlue.c JXRMeta.c JXRGluePFC.c JXRGlueJxr.c)
JXR_MAKE_OBJ(GLUE)
set(SRC_TEST JXRTest.c JXRTestBmp.c JXRTestHdr.c JXRTestPnm.c JXRTestTif.c JXRTestYUV.c)
JXR_MAKE_OBJ(TEST)

add_library(jxrglue STATIC ${OBJ_GLUE} ${OBJ_TEST})
set_property(TARGET jxrglue
  PROPERTY COMPILE_DEFINITIONS __ANSI__ DISABLE_PERF_MEASUREMENT ${DEF_ENDIAN}
)
set_property(TARGET jxrglue PROPERTY COMPILE_FLAGS -w)
set_property(TARGET jxrglue PROPERTY VERSION ${VERSION})
set_property(TARGET jxrglue PROPERTY SOVERSION ${SOVERSION})
install(TARGETS jxrglue
  EXPORT JXRLibTargets
  RUNTIME DESTINATION ${JXRLIB_INSTALL_BINDIR} COMPONENT Applications
  LIBRARY DESTINATION ${JXRLIB_INSTALL_LIBDIR} COMPONENT Libraries
)

target_link_libraries(jxrglue PRIVATE jpegxr)

# Enc app files
set(ENCAPP JxrEncApp)
add_executable(${ENCAPP} ${DIR_EXEC}/${ENCAPP}.c)
set_property(TARGET ${ENCAPP}
  PROPERTY COMPILE_DEFINITIONS __ANSI__ DISABLE_PERF_MEASUREMENT ${DEF_ENDIAN}
)
set_property(TARGET ${ENCAPP} PROPERTY COMPILE_FLAGS -w)
target_link_libraries(${ENCAPP} jxrglue) # jpegxr)
install(TARGETS ${ENCAPP} RUNTIME DESTINATION ${JXRLIB_INSTALL_BINDIR})

# Dec app files
set(DECAPP JxrDecApp)
add_executable(${DECAPP} ${DIR_EXEC}/${DECAPP}.c)
set_property(TARGET ${DECAPP}
  PROPERTY COMPILE_DEFINITIONS __ANSI__ DISABLE_PERF_MEASUREMENT ${DEF_ENDIAN}
)
set_property(TARGET ${DECAPP} PROPERTY COMPILE_FLAGS -w)
target_link_libraries(${DECAPP} jxrglue) # jpegxr)
install(TARGETS ${DECAPP} RUNTIME DESTINATION ${JXRLIB_INSTALL_BINDIR})

# Generate pkgconfig file
include(JoinPaths.cmake)
join_paths(JXRLIB_PKGCONF_LIBDIR "\${prefix}" "${JXRLIB_INSTALL_LIBDIR}")
join_paths(JXRLIB_PKGCONF_INCLUDEDIR "\${prefix}" "${JXRLIB_INSTALL_INCLUDEDIR}")
configure_file(
    "${PROJECT_SOURCE_DIR}/jxrlib.pc.in"
    ${PROJECT_BINARY_DIR}/jxrlib.pc
    @ONLY
)
install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
    DESTINATION ${JXRLIB_INSTALL_LIBDIR}/pkgconfig
)

# install rules
install(FILES jxrgluelib/JXRGlue.h jxrgluelib/JXRMeta.h jxrtestlib/JXRTest.h
  image/sys/windowsmediaphoto.h
  DESTINATION ${JXRLIB_INSTALL_INCLUDEDIR} COMPONENT Headers
)
install(DIRECTORY common/include/ DESTINATION ${JXRLIB_INSTALL_INCLUDEDIR}
  FILES_MATCHING PATTERN "*.h"
)
