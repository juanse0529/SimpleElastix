cmake_minimum_required ( VERSION 3.0 FATAL_ERROR )

# Explicitly add INCREMENTAL linking option to command lines.
# http://www.cmake.org/pipermail/cmake/2010-February/035174.html
SET(MSVC_INCREMENTAL_DEFAULT ON)

project ( SimpleITK )

cmake_policy( VERSION 3.0 )

foreach(p
    CMP0042 # CMake 3.0
    CMP0063 # CMake 3.3.2
    )
  if(POLICY ${p})
    cmake_policy(SET ${p} NEW)
  endif()
endforeach()

# Include extra CMake files
list(INSERT CMAKE_MODULE_PATH 0 "${CMAKE_CURRENT_SOURCE_DIR}/CMake")

#-----------------------------------------------------------------------------
# Version information
include(Version.cmake)

set(SimpleITK_VERSION "${SimpleITK_VERSION_MAJOR}.${SimpleITK_VERSION_MINOR}")
if(DEFINED SimpleITK_VERSION_PATCH)
  set(SimpleITK_VERSION "${SimpleITK_VERSION}.${SimpleITK_VERSION_PATCH}")
  if(DEFINED SimpleITK_VERSION_TWEAK)
    set(SimpleITK_VERSION "${SimpleITK_VERSION}.${SimpleITK_VERSION_TWEAK}")
  endif()
endif()

if(DEFINED SimpleITK_VERSION_RC)
  set(SimpleITK_VERSION "${SimpleITK_VERSION}${SimpleITK_VERSION_RC}")
endif()
if(DEFINED SimpleITK_VERSION_POST)
  set(SimpleITK_VERSION "${SimpleITK_VERSION}.post${SimpleITK_VERSION_POST}")
elseif(DEFINED SimpleITK_VERSION_DEV)
  set(SimpleITK_VERSION "${SimpleITK_VERSION}.dev${SimpleITK_VERSION_DEV}")
endif()

option( SimpleITK_BUILD_DISTRIBUTE "Remove '-g#####' from version. ( for official distribution only )" OFF )
mark_as_advanced( SimpleITK_BUILD_DISTRIBUTE )
if( NOT SimpleITK_BUILD_DISTRIBUTE )
  set(SimpleITK_VERSION "${SimpleITK_VERSION}-g${SimpleITK_VERSION_HASH}")
endif()

message(STATUS "Building SimpleITK version \"${SimpleITK_VERSION}\"")


#-----------------------------------------------------------
# Check and set required flags for compilation
# This must be done before any other try compiles based tests are
# done.
include(sitkCheckRequiredFlags)
include(sitkCompilerWarningsSettings)


find_package(ITK REQUIRED )
#we require certain packages be turned on in ITK
include(sitkCheckForITKModuleDependencies)

if(ITK_FOUND)
  # NOTE: We are purposely not calling UseITK yet. However, we must make
  # sure the requred compilation and linker flags are set. Since, we
  # are trying to encapsulate ITK, we need to very carefully control
  # in access to the headers and libraries, hence each SimpleITK
  # library will call UseITK.

  # Add compiler flags needed to use ITK.
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${ITK_REQUIRED_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${ITK_REQUIRED_CXX_FLAGS}")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${ITK_REQUIRED_LINK_FLAGS}")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${ITK_REQUIRED_LINK_FLAGS}")
  set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${ITK_REQUIRED_LINK_FLAGS}")

  link_directories( "${ITK_LIBRARY_DIRS}")
endif()

#-----------------------------------------------------------
# Configure Elastix
find_package( OpenMP QUIET )
if( OPENMP_FOUND )
  set(SimpleITK_REQUIRED_C_FLAGS "${SimpleITK_REQUIRED_C_FLAGS} ${OpenMP_C_FLAGS}")
  set(SimpleITK_REQUIRED_CXX_FLAGS "${SimpleITK_REQUIRED_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

# Find UseElastix.cmake 
if( NOT EXISTS ${ELASTIX_USE_FILE} )
  set( ELASTIX_USE_FILE ${ELASTIX_DIR}/UseElastix.cmake )
endif()

if( NOT EXISTS ${ELASTIX_USE_FILE} )
  # UseElastix.cmake path NOT passed to build. Ask user to specify custom elastix install or use the SuperBuild.
  set( ELASTIX_DIR "" CACHE PATH "Path to elastix build folder" )
  message(FATAL_ERROR "Could not find UseElastix.cmake. Point ELASTIX_DIR to folder containing UseElastix.cmake or use SuperBuild.")
endif()

include( ${ELASTIX_USE_FILE} )

# Check if libraries are available
if( ELASTIX_BUILD_EXECUTABLE )
  message(FATAL_ERROR "SimpleElastix requires elastix libraries, please set ELASTIX_BUILD_EXECUTABLE=OFF and rebuild elastix")
endif()

#----------------------------------------------------------
# Set flags and directories
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${SimpleITK_REQUIRED_C_FLAGS}")
set(CMAKE_CXX_FLAGS "${CXX_ADDITIONAL_WARNING_FLAGS} ${SimpleITK_REQUIRED_CXX_FLAGS}")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${SimpleITK_REQUIRED_LINK_FLAGS}")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${SimpleITK_REQUIRED_LINK_FLAGS}")
set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${SimpleITK_REQUIRED_LINK_FLAGS}")

#----------------------------------------------------------
# Place all options to go into sitkConfigure.h here
option(BUILD_SHARED_LIBS "Build SimpleITK ITK with shared libraries. This does not effect wrapped languages." OFF)
set(SITK_BUILD_SHARED_LIBS ${BUILD_SHARED_LIBS})

option(SITK_EXPLICIT_INSTANTIATION "Enable an ITK static library of explicitly instantiated templates." ON)
if ( MSVC AND SITK_BUILD_SHARED_LIBS )
  set( SITK_SimpleITKExplit_STATIC 1 )
endif()

set( SITK_INT64_PIXELIDS_DEFAULT ON )
if( MSVC )
  # See http://www.cmake.org/cmake/help/v2.8.10/cmake.html#variable:MSVC_VERSION
  # and https://en.wikipedia.org/wiki/Microsoft_Visual_Studio#Version_history
  #   1500 = VS  9.0 (Visual Studio 2008)
  #   1600 = VS 10.0 (Visual Studio 2010)
  #   1700 = VS 11.0 (Visual Studio 2012)
  if(MSVC_VERSION VERSION_LESS 1600)
    # with this option on the linker runs out of memory
    set( SITK_INT64_PIXELIDS_DEFAULT OFF )
  endif()
endif()
option( SITK_INT64_PIXELIDS "Instantiate 64-bit integer pixels, including unsigned, vector and label maps." ${SITK_INT64_PIXELIDS_DEFAULT} )
# Setting this option will shorten compile times by reducing the
# amount of pixel types and generated code.
# NB: this option should _NOT_ be used for releases!
option ( SITK_EXPRESS_INSTANTIATEDPIXELS "Instantiate very few pixel types ( for use for development only )" OFF )
mark_as_advanced ( SITK_EXPRESS_INSTANTIATEDPIXELS )

option( SITK_4D_IMAGES "Add Image and I/O support for four spatial dimensions." OFF )
mark_as_advanced( SITK_4D_IMAGES )


include( sitkForbidDownloadsOption )

# Setup build locations.
if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)
endif()
if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
endif()
if(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
endif()

set ( SimpleITK_INCLUDE_DIRS
      ${CMAKE_SOURCE_DIR}/Code/Common/include
      ${CMAKE_SOURCE_DIR}/Code/Registration/include
      ${CMAKE_SOURCE_DIR}/Code/BasicFilters/include
      ${CMAKE_SOURCE_DIR}/Code/IO/include
      ${CMAKE_SOURCE_DIR}/Code/Elastix/include
      ${CMAKE_BINARY_DIR}/Code/BasicFilters/include
      ${CMAKE_BINARY_DIR}/Code/Common/include
      )

set ( SimpleITK_LIBRARIES SimpleITKCommon SimpleITKIO SimpleITKRegistration SimpleElastix SimpleTransformix )

if (SITK_EXPLICIT_INSTANTIATION)
  list ( APPEND SimpleITK_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/Code/Explicit/include )
  # This library is linked privately to SimpleITK libraries and is not
  # part of the public link interface
  #list ( APPEND SimpleITK_LIBRARIES  SimpleITKExplicit )
endif()

set( SimpleITK_LIBRARIES ${SimpleITK_LIBRARIES}  CACHE INTERNAL "" )

# Be sure to clear these each time
set ( GENERATED_TEST_LIST "" CACHE INTERNAL "" )
set ( GENERATED_FILTER_LIST "" CACHE INTERNAL "" )
set ( GENERATED_TEST_SOURCE_LIST "" CACHE INTERNAL "" )

# Create cached list of all template components
file( GLOB template_components
      ${SimpleITK_SOURCE_DIR}/TemplateComponents/*.h.in
      ${SimpleITK_SOURCE_DIR}/TemplateComponents/*.cxx.in
    )
set ( template_components ${template_components} CACHE INTERNAL "" )


#------------------------------------------------------------------------------
# Variables for use in install rules:

if(NOT SimpleITK_INSTALL_RUNTIME_DIR)
  set(SimpleITK_INSTALL_RUNTIME_DIR "bin")
endif()
if(NOT SimpleITK_INSTALL_LIBRARY_DIR)
  set(SimpleITK_INSTALL_LIBRARY_DIR "lib")
endif()
if(NOT SimpleITK_INSTALL_ARCHIVE_DIR)
  set(SimpleITK_INSTALL_ARCHIVE_DIR "lib")
endif()
if(NOT SimpleITK_INSTALL_INCLUDE_DIR)
  set(SimpleITK_INSTALL_INCLUDE_DIR "include/SimpleITK-${SimpleITK_VERSION_MAJOR}.${SimpleITK_VERSION_MINOR}")
endif()
if(NOT SimpleITK_INSTALL_DOC_DIR)
  set(SimpleITK_INSTALL_DOC_DIR share/doc/SimpleITK-${SimpleITK_VERSION_MAJOR}.${SimpleITK_VERSION_MINOR})
endif()
if(NOT SimpleITK_INSTALL_PACKAGE_DIR)
  set(SimpleITK_INSTALL_PACKAGE_DIR "lib/cmake/SimpleITK-${SimpleITK_VERSION_MAJOR}.${SimpleITK_VERSION_MINOR}")
endif()

function(sitk_install_exported_target tgt)
  # Remove the build tree's SimpleITKTargets file if this is the first call:
  get_property(first_time GLOBAL PROPERTY SimpleITK_FIRST_EXPORTED_TARGET)
  if(NOT first_time)
    file(REMOVE ${CMAKE_BINARY_DIR}/SimpleITKTargets.cmake)
    set_property(GLOBAL PROPERTY SimpleITK_FIRST_EXPORTED_TARGET 1)
  endif()

  set_property(TARGET ${tgt} PROPERTY VERSION 1)
  set_property(TARGET ${tgt} PROPERTY SOVERSION 1)
  set_property(TARGET ${tgt} PROPERTY
    OUTPUT_NAME ${tgt}-${SimpleITK_VERSION_MAJOR}.${SimpleITK_VERSION_MINOR})

  export(TARGETS ${tgt}
    APPEND FILE "${CMAKE_BINARY_DIR}/SimpleITKTargets.cmake"
    )

  install(TARGETS ${tgt}
    EXPORT SimpleITKTargets
    RUNTIME DESTINATION ${SimpleITK_INSTALL_RUNTIME_DIR}
    LIBRARY DESTINATION ${SimpleITK_INSTALL_LIBRARY_DIR}
    ARCHIVE DESTINATION ${SimpleITK_INSTALL_ARCHIVE_DIR}
    )
endfunction()

include(sitkStripOption)

#------------------------------------------------------------------------------
# These are some system specific compiler options needed to build SimpleITK

include(CheckCXXCompilerFlag)

if( NOT DEFINED CMAKE_POSITION_INDEPENDENT_CODE )
  set( CMAKE_POSITION_INDEPENDENT_CODE 1 )
endif()


if( NOT DEFINED CMAKE_C_VISIBILITY_PRESET )
  set(CMAKE_C_VISIBILITY_PRESET hidden)
endif()

if( NOT DEFINED CMAKE_CXX_VISIBILITY_PRESET )
  set(CMAKE_CXX_VISIBILITY_PRESET hidden)
endif()

if( NOT DEFINED CMAKE_VISIBILITY_INLINES_HIDDEN )
  set( CMAKE_VISIBILITY_INLINES_HIDDEN 1 )
endif()


#-----------------------------------------------------------
# Place all checks and try compile variable for sitkConfigure.h here

include(sitkCheckCXX11)


include(CheckIncludeFileCXX)


# check for stdint
# This generates a configuration error if the compiler is not supported
check_include_file_cxx( stdint.h SITK_HAS_STDINT_H )

if(NOT STDINT_HAVE_STDINT_H)
  if( WIN32 )
    if(MSVC)
      list(APPEND SimpleITK_INCLUDE_DIRS
        "${SimpleITK_SOURCE_DIR}/Utilities/C99")
      add_subdirectory("${SimpleITK_SOURCE_DIR}/Utilities/C99")
    endif()
  endif()
endif()

include_directories( ${SimpleITK_INCLUDE_DIRS} )


#------------------------------------------------------------------------------
# assemble a list of important documentation from Simple ITK and ITK

set ( SimpleITK_DOC_FILES
  "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE"
  "${CMAKE_CURRENT_SOURCE_DIR}/NOTICE"
  "${CMAKE_CURRENT_SOURCE_DIR}/README.md"
  )

# add important files from ITK

# if using from build tree then ITK_USE_FILE will be
# ITK_SOURCE_DIR/CMake/UseITK.cmake. The needed docs should be in the
# root of the source tree
get_filename_component( _itk_root "${ITK_USE_FILE}" PATH )
get_filename_component( _itk_root "${_itk_root}" PATH )
set( _itk_source_path "${_itk_root}" )

get_filename_component( _itk_root "${_itk_root}" PATH )
get_filename_component( _itk_root "${_itk_root}" PATH )
set( _itk_doc_path "${_itk_root}/share/doc/ITK-${ITK_VERSION_MAJOR}.${ITK_VERSION_MINOR}" )

if( EXISTS "${_itk_source_path}/NOTICE" )
  set( ITK_DOC_FILES
    # "${_itk_source_path}/LICENSE" SimpleITK has same license file
    "${_itk_source_path}/NOTICE"
    "${_itk_source_path}/README.txt"
    )
elseif( EXISTS "${_itk_doc_path}/NOTICE" )
  set( ITK_DOC_FILES
    # "${_itk_doc_path}/LICENSE" SimpleITK has same license file
    "${_itk_doc_path}/NOTICE"
    "${_itk_doc_path}/README.txt"
    )
else()
  message( "doc_path: ${_itk_doc_path}" )
  message( WARNING "Unable to file ITK's NOTICE File!\n This file will not be included in SimpleITK packaging!" )
endif()


foreach(_f IN LISTS ITK_DOC_FILES)
  get_filename_component( _name "${_f}" NAME )
  set( _o "${CMAKE_CURRENT_BINARY_DIR}/ITK-${ITK_VERSION_MAJOR}.${ITK_VERSION_MINOR}-${_name}" )
  configure_file( "${_f}"  "${_o}" )
  list( APPEND SimpleITK_DOC_FILES "${_o}" )
endforeach()


#------------------------------------------------------------------------------
# Set up Documentation
include(${SimpleITK_SOURCE_DIR}/Utilities/Doxygen/Doxygen.cmake)

#------------------------------------------------------------------------------
# Set up wrapping.
#
# Use CMake file which present options for wrapped languages, and finds languages as needed
#
include(sitkLanguageOptions)

#------------------------------------------------------------------------------
# Ensure that development strips have been setup
include(sitkCheckSourceTree)

#-----------------------------------------------------------------------------
if(NOT EXISTS "${SimpleITK_SOURCE_DIR}/.ExternalData/README.rst")
  # This file is always present in version-controlled source trees
  # so we must have been extracted from a source tarball with no
  # data objects needed for testing.  Turn off tests by default
  # since enabling them requires network access or manual data
  # store configuration.
  option(BUILD_TESTING "Build the testing tree." OFF)
endif()

#------------------------------------------------------------------------------
# set things up for testing, this configuration needs to occour before
# we enter the sub-directories
include(CTest)

#------------------------------------------------------------------------------
# Go to subdirectories
add_subdirectory ( Utilities )
add_subdirectory ( Code )
add_subdirectory ( Wrapping )

#------------------------------------------------------------------------------
# ITK uses KWStyle for checking the coding style
include(${SimpleITK_SOURCE_DIR}/Utilities/KWStyle/KWStyle.cmake)

#------------------------------------------------------------------------------
# set things up for testing, if enabled

if ( BUILD_TESTING )

  include( sitkAddTest )

  file( GLOB_RECURSE content_links
    RELATIVE "${SimpleITK_SOURCE_DIR}" "Testing/Data/*.md5" )
  foreach(link ${content_links})
    string( REGEX REPLACE "\\.md5$" "" link ${link} )
    ExternalData_Expand_Arguments( SimpleITKData
      link_location
      DATA{${link}}
      )
  endforeach()

  enable_testing()
  add_subdirectory ( Testing )

  option ( RUN_LONG_TESTS "Run long tests.  Some tests have been identified as long running, should these be run?" OFF )
  mark_as_advanced(RUN_LONG_TESTS)
  set(BUILDNAME "${BUILDNAME}" CACHE STRING "Name of build on the dashboard")
  configure_file(CMake/CTestCustom.cmake.in CTestCustom.cmake)
endif()

# optional examples directory with testing
option(BUILD_EXAMPLES "Build the Examples directory." ON)

if(BUILD_EXAMPLES)
  add_subdirectory ( Examples )
endif()


#------------------------------------------------------------------------------
# Options for documentation
option ( BUILD_DOCUMENTS "Build the Documentation subdirectory" OFF )

if ( BUILD_DOCUMENTS )
  add_subdirectory ( Documentation )
endif()


#------------------------------------------------------------------------------
# Configure SimpleITKConfig.cmake for the build tree.
set(SimpleITKConfig_TREE "build")
set(SimpleITKConfig_CODE "")
set(SimpleITKConfig_INCLUDE_DIRS ${SimpleITK_INCLUDE_DIRS})

configure_file(SimpleITKConfig.cmake.in
               SimpleITKConfig.cmake @ONLY IMMEDIATE)

#------------------------------------------------------------------------------
# Configure SimpleITKConfig.cmake for the install tree.
set(SimpleITKConfig_TREE "install")

set(SimpleITKConfig_CODE "set(SimpleITK_INSTALL_PREFIX \"\${_SimpleITKConfig_DIR}\")")
# Construct the proper number of get_filename_component(... PATH)
# calls to compute the installation prefix.
string(REGEX REPLACE "/" ";" _count "${SimpleITK_INSTALL_PACKAGE_DIR}")
foreach(p ${_count})
  set(SimpleITKConfig_CODE "${SimpleITKConfig_CODE}
get_filename_component(SimpleITK_INSTALL_PREFIX \"\${SimpleITK_INSTALL_PREFIX}\" PATH)")
endforeach(p)
set(SimpleITKConfig_CODE "${SimpleITKConfig_CODE}\n")

set(SimpleITKConfig_INCLUDE_DIRS
  "\${SimpleITK_INSTALL_PREFIX}/${SimpleITK_INSTALL_INCLUDE_DIR}")

configure_file(SimpleITKConfig.cmake.in
               CMakeFiles/SimpleITKConfig.cmake @ONLY IMMEDIATE)

#------------------------------------------------------------------------------
# Help other projects use SimpleITK
configure_file(UseSimpleITK.cmake.in
               UseSimpleITK.cmake COPYONLY IMMEDIATE)

configure_file(SimpleITKConfigVersion.cmake.in
               SimpleITKConfigVersion.cmake @ONLY IMMEDIATE)


#------------------------------------------------------------------------------
# INSTALLATION

install(FILES ${SimpleITK_BINARY_DIR}/CMakeFiles/SimpleITKConfig.cmake
              ${SimpleITK_BINARY_DIR}/SimpleITKConfigVersion.cmake
              ${SimpleITK_BINARY_DIR}/UseSimpleITK.cmake
  DESTINATION ${SimpleITK_INSTALL_PACKAGE_DIR}
  COMPONENT Development)

install(EXPORT SimpleITKTargets
  DESTINATION ${SimpleITK_INSTALL_PACKAGE_DIR}
  COMPONENT Development)

file( GLOB __files
  ${CMAKE_SOURCE_DIR}/Code/BasicFilters/include/*.h
  ${CMAKE_SOURCE_DIR}/Code/BasicFilters/include/*.hxx
  ${CMAKE_SOURCE_DIR}/Code/Common/include/*.h
  ${CMAKE_SOURCE_DIR}/Code/Common/include/*.hxx
  ${CMAKE_SOURCE_DIR}/Code/Elastix/include/*.h
  ${CMAKE_SOURCE_DIR}/Code/Elastix/include/*.hxx
  ${CMAKE_SOURCE_DIR}/Code/IO/include/*.h
  ${CMAKE_SOURCE_DIR}/Code/IO/include/*.hxx
  ${CMAKE_SOURCE_DIR}/Code/Registration/include/*.h
  ${CMAKE_SOURCE_DIR}/Code/Registration/include/*.hxx
  )

install(FILES ${__files}
  DESTINATION ${SimpleITK_INSTALL_INCLUDE_DIR}
  COMPONENT Development)

install(
  DIRECTORY
    ${CMAKE_SOURCE_DIR}/Code/Common/include/Ancillary
    ${CMAKE_SOURCE_DIR}/Code/Common/include/nsstd
  DESTINATION
    ${SimpleITK_INSTALL_INCLUDE_DIR}/
  COMPONENT Development
  FILES_MATCHING PATTERN "*.h"
)

install(FILES ${SimpleITK_DOC_FILES} DESTINATION "${SimpleITK_INSTALL_DOC_DIR}" COMPONENT Runtime)

install( DIRECTORY "${ELASTIX_LINK_DIRECTORIES}"
         DESTINATION ${SimpleITK_INSTALL_INCLUDE_DIR} 
         COMPONENT Development 
         FILES_MATCHING PATTERN "*ix*" )

#------------------------------------------------------------------------------
# CPack

set(CPACK_SOURCE_IGNORE_FILES  "${ITK_MODULES_DISABLED_CPACK};/\\\\.git")
set(CPACK_PACKAGE_VERSION_MAJOR "${SimpleITK_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${SimpleITK_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${SimpleITK_VERSION_PATCH}")


include( CPack )

if(BUILD_TESTING)
  if(COMMAND ExternalData_Add_Target )
    ExternalData_Add_Target( SimpleITKData )
  endif()
endif()
