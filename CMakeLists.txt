########################################################
# cmake file for building Marlin example Package
CMAKE_MINIMUM_REQUIRED(VERSION 2.6 FATAL_ERROR)
########################################################

# project name
PROJECT( Eutelescope )

# project version; before changing this, read README.release_procedure
SET( ${PROJECT_NAME}_VERSION_MAJOR 1 )
SET( ${PROJECT_NAME}_VERSION_MINOR 0 )
SET( ${PROJECT_NAME}_VERSION_PATCH 0 )

# These switches activate/deactivate compiler switches such as the
# inclusion of debugging symbols and code optimizations. For actual
# data analysis we recommend setting CMAKE_BUILD_TYPE to "Release"
# while for development "RelWithDebInfo" is best used. For actual
# debugging e.g. using gdb, please consider "Debug" which will turn
# off all (!) optimizations but will be "gdb-friendlier"
# !RELEASE: comment out the following line and comment in either one of the release ones
# SET( CMAKE_BUILD_TYPE Debug ) # use for debugging with e.g. gdb: add. warnings, no optimizations, debug symbols
SET( CMAKE_BUILD_TYPE RelWithDebInfo ) # useful for normal development, runs faster but compilation takes longer: gcc optimizations (-O2) with debug symbols and add. warnings
#SET( CMAKE_BUILD_TYPE Release ) # for production: full gcc optimization (-O3), no debug symbols, no debug messages


### DEPENDENCIES ############################################################

FIND_PACKAGE( ILCUTIL COMPONENTS ILCSOFT_CMAKE_MODULES REQUIRED ILCTEST)

# load default settings from ILCSOFT_CMAKE_MODULES
INCLUDE( ilcsoft_default_settings )

# additional packages to be searched for by cmake
LIST( APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake )

FIND_PACKAGE( Marlin 1.0 REQUIRED )
FIND_PACKAGE( GSL )
FIND_PACKAGE( AIDA )
FIND_PACKAGE( ROOT COMPONENTS Minuit Geom )
FIND_PACKAGE( LCCD  REQUIRED )               

# search for Eigen (linear algebra) library
FIND_PACKAGE( Eigen3 REQUIRED )
# include them as SYSTEM include directories, this will supress all warnings from them
INCLUDE_DIRECTORIES( SYSTEM ${EIGEN3_INCLUDE_DIR} )

# General Broken Line track fitter as shared lib
FIND_PACKAGE( GBL )

FOREACH( pkg Marlin MarlinUtil GSL AIDA ROOT LCCD GBL )
    IF( ${pkg}_FOUND )
        # include as "system" libraries: gcc will be less verbose w.r.t. warnings
        INCLUDE_DIRECTORIES( SYSTEM ${${pkg}_INCLUDE_DIRS} )
        LINK_LIBRARIES( ${${pkg}_LIBRARIES} )
        ADD_DEFINITIONS ( ${${pkg}_DEFINITIONS} )
    ELSE()
        MESSAGE( STATUS "${pkg} not found" )
    ENDIF()
ENDFOREACH()


### LIBRARY #################################################################

#GBL
IF( GBL_FOUND )
    ADD_DEFINITIONS( "-DUSE_GBL" )
    GET_FILENAME_COMPONENT( GBL_LIBRARY_FULL_PATH ${GBL_GBL_LIBRARY} REALPATH )
    INSTALL( FILES ${GBL_LIBRARY_FULL_PATH} DESTINATION lib )
ELSE()
    MESSAGE( STATUS "WARNING: failed to configure Eutelescope with GBL!!" )
ENDIF()

#AIDA
IF( AIDA_FOUND )
    ADD_DEFINITIONS( "-DUSE_AIDA" )
ENDIF()

#GSL
IF( GSL_FOUND )
  ADD_DEFINITIONS( "-DUSE_GSL" )
ENDIF()

#non-optional dependencies
ADD_DEFINITIONS( "-DUSE_MARLIN" )
ADD_DEFINITIONS( "-DUSE_GEAR" )
ADD_DEFINITIONS( "-DUSE_CLHEP" )

# ---------------------------------------------------------------------------

# include directories
INCLUDE_DIRECTORIES( SYSTEM ./external )
INCLUDE_DIRECTORIES( ./eutelescope/libraries/include )
INCLUDE_DIRECTORIES( ./eutelescope/interfaces/pixel/include )
INCLUDE_DIRECTORIES( ./eutelescope/interfaces/cluster/include )
INCLUDE_DIRECTORIES( ./processors/include )
INCLUDE_DIRECTORIES( ./processors/include/legacy )
INCLUDE_DIRECTORIES( ./processors/include/alibava )
INCLUDE_DIRECTORIES( ./processors/include/cms )
INCLUDE_DIRECTORIES( ./geometries/include/ )
INCLUDE_DIRECTORIES( ./external/millepede2/$ENV{MILLEPEDEII_VERSION} )

# definitions to pass to the compiler
ADD_DEFINITIONS( "-Wno-effc++ -std=c++14 -Wno-shadow -Wall -Wdisabled-optimization \
		 -Wold-style-cast -pedantic -Wextra -Wcast-align -Wcast-qual \
		 -Wuseless-cast -Wzero-as-null-pointer-constant -Wundef \
		 -Wctor-dtor-privacy -Wredundant-decls -Wdeprecated -Wformat-security \
		 -Wmissing-declarations -Wmissing-include-dirs -Wmissing-include-dirs \
		 -Wswitch-default -Wno-sign-promo -Werror" )

# debuging warnings and switches for CMAKE_BUILD_TYPE=DEBUG
SET( CMAKE_CXX_FLAGS_DEBUG "-O0 -g -fno-inline -fdiagnostics-show-option -Weffc++ \
     -Wcast-align -Wcast-qual -Wctor-dtor-privacy -Wdisabled-optimization -Wformat=2 \
     -Winit-self -Wmissing-include-dirs -Wold-style-cast -Woverloaded-virtual \
     -Wredundant-decls -Wsign-promo -Wstrict-null-sentinel -Wswitch-default -Wundef"
     CACHE STRING "Debug options." FORCE )
# debuging warnings and switches for CMAKE_BUILD_TYPE=RELWITHDEBINFO
SET( CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g -fdiagnostics-show-option -Weffc++ \
     -Wcast-align -Wcast-qual -Wdisabled-optimization -Winit-self -Wmissing-include-dirs \
     -Wredundant-decls -Wsign-promo -Wstrict-null-sentinel -Wswitch-default -Wundef"
     CACHE STRING "Debug options." FORCE )


############ Add libraries ########################################

# Main EUTelescope Library
SET( libname ${PROJECT_NAME} )
AUX_SOURCE_DIRECTORY( ${CMAKE_CURRENT_SOURCE_DIR}/eutelescope/libraries/src library_sources )
AUX_SOURCE_DIRECTORY( ${CMAKE_CURRENT_SOURCE_DIR}/eutelescope/interfaces/pixel/src library_sources )
AUX_SOURCE_DIRECTORY( ${CMAKE_CURRENT_SOURCE_DIR}/eutelescope/interfaces/cluster/src library_sources )

ADD_SHARED_LIBRARY( ${libname} ${library_sources} ${CMAKE_CURRENT_SOURCE_DIR}/external/millepede2/$ENV{MILLEPEDEII_VERSION}/Mille.cc )
INSTALL_SHARED_LIBRARY( ${libname} DESTINATION lib )

OPTION( BUILD_PROCESSORS "Build the EUTelescope Marlin Processors" ON )
IF( BUILD_PROCESSORS )
  # Processor Library
  SET( processors EutelProcessors )
  AUX_SOURCE_DIRECTORY( ${CMAKE_CURRENT_SOURCE_DIR}/processors/src processor_sources )
  AUX_SOURCE_DIRECTORY( ${CMAKE_CURRENT_SOURCE_DIR}/processors/src/legacy processor_sources )
  AUX_SOURCE_DIRECTORY( ${CMAKE_CURRENT_SOURCE_DIR}/processors/src/alibava processor_sources )
  AUX_SOURCE_DIRECTORY( ${CMAKE_CURRENT_SOURCE_DIR}/processors/src/cms processor_sources )
  ADD_SHARED_LIBRARY( ${processors} ${processor_sources} )
  INSTALL_SHARED_LIBRARY( ${processors} DESTINATION lib )

  # FIXME: making here a copy of shared library because MARLIN_DLL expects libEutelReaders.so; this hack is here only for backward compatibility
  AUX_SOURCE_DIRECTORY( ${CMAKE_CURRENT_SOURCE_DIR}/processors/src/legacy dummy_sources )
  ADD_SHARED_LIBRARY( EutelReaders  ${dummy_sources} )
  INSTALL_SHARED_LIBRARY( EutelReaders DESTINATION lib )
ENDIF()

# Pixel Geometry Shared Libraries
FILE(GLOB IMPL_SRCS ${CMAKE_CURRENT_SOURCE_DIR}/geometries/src/*.cc)

FOREACH(impl_src ${IMPL_SRCS})
    # Extract FileName from FileName.cc
    GET_FILENAME_COMPONENT(basename ${impl_src} NAME_WE) 
    ADD_LIBRARY( ${basename} SHARED ${impl_src} )
	INSTALL_SHARED_LIBRARY( ${basename} DESTINATION lib )
ENDFOREACH()  

# used for alignment if Eutelescope was build with ROOT support
IF( ROOT_FOUND AND ROOT_MINUIT_FOUND )
    TARGET_LINK_LIBRARIES( ${libname} ${ROOT_MINUIT_LIBRARY} )
ENDIF()

IF( ROOT_FOUND AND ROOT_GEOM_FOUND )
    TARGET_LINK_LIBRARIES( ${libname} ${ROOT_GEOM_LIBRARY} )
ENDIF()

# display some variables and write them to cache
DISPLAY_STD_VARIABLES()

# create some useful symbolic links in bin directory at install time
INSTALL( CODE
    	 "EXECUTE_PROCESS(COMMAND ln -sf ${PROJECT_SOURCE_DIR}/jobsub/jobsub.py ${PROJECT_SOURCE_DIR}/bin/jobsub)
     	  EXECUTE_PROCESS(COMMAND ln -sf ${PROJECT_SOURCE_DIR}/jobsub/jobsub.py ${PROJECT_SOURCE_DIR}/bin/jobsub.py)"
)

# Enable testing
ENABLE_TESTING()

# Include example configurations:
OPTION( ctestFlag "Build all ctests." OFF )
IF( ctestFlag )
  ADD_SUBDIRECTORY( ctest )
ENDIF()

OPTION( utest "Build all tests." OFF )
IF(utest)
  ADD_SUBDIRECTORY( unittests )
ENDIF()
  
#  _            _       
# | |_ ___  ___| |_ ___ 
# | __/ _ \/ __| __/ __|
# | ||  __/\__ \ |_\__ \
#  \__\___||___/\__|___/
#=============================================================================                      
# EUTelescope uses the CMake/CTest framework for daily automated,
#data-driven tests to constantly verify the validity of the output. The
#basis for the tests are the examples provided with the software
#together with a real data sample and the expected output in form of
#root files.
#
#If you are a user of EUTelescope and you have already taken data and
#performed you analysis, we would be very interested in obtaining a
#data sample with the corresponding analysis results including the
#EUTelescope configuration used so that we can include your analysis in
#the regular testing. This guarantees that your analysis will be well
#supported in future releases of EUTelescope and it might serve as a
#good starting point for others.
#
#The test can also be used by any developer who wishes to verify
#changes made to the EUTelescope code. For a short how-to see
#README.tests; for more detailed information see the CMake manual or
#web page (http://www.cmake.org/Wiki/CMake/Testing_With_CTest).

ENABLE_TESTING()
INCLUDE( CTest )


# ----------------------------------------------------------------------------
# TEST IF ENVIRONMENT IS OK FOR RUNNING TESTS
# ----------------------------------------------------------------------------

IF( NOT DEFINED ENV{EUTELESCOPE} OR NOT DEFINED ENV{MARLIN} )
	MESSAGE( "trying to load build and run environment for EUTelescope" )

  	IF( DEFINED ENV{ILCSOFT} )
        EXECUTE_PROCESS( COMMAND sh -c " [ -f $ILCSOFT/Eutelscope/trunk/build_env.sh ] && . $ILCSOFT/Eutelescope/trunk/build_env.sh" )

		IF( NOT DEFINED ENV{EUTELESCOPE} OR NOT DEFINED ENV{MARLIN} )
        	# try to load build_env.sh environment - if running 'make' from 'build' directory, it should be located in '../'
        	EXECUTE_PROCESS( COMMAND sh -c " [ -f '../build_env.sh' ] && . ../build_env.sh" )
	
			IF( DEFINED ENV{EUTELESCOPE} AND DEFINED ENV{MARLIN} )
	    		MESSAGE( "EUTELESCOPE environment automatically loaded from ../build_env.sh" )
			ENDIF()
   		ENDIF()
	ENDIF()
ENDIF()

# ----------------------------------------------------------------------------
# ACTUAL TESTS BEGIN HERE
# ----------------------------------------------------------------------------

# check environment
IF( DEFINED ENV{EUTELESCOPE} AND DEFINED ENV{MARLIN})

# use valgrind for dynamic analysis
SET( CTEST_MEMORYCHECK_COMMAND "/usr/bin/valgrind")
SET( CTEST_MEMORYCHECK_COMMAND_OPTIONS "-q --num-callers=50 --tool=memcheck \
     --track-origins=yes --trace-children=yes --leak-check=full --leak-resolution=low \
     --suppressions=$ENV{ROOTSYS}/etc/valgrind-root.supp \
     --suppressions=$ENV{EUTELESCOPE}/test/valgrind/valgrind-python.supp")

# User Options to be included in DartConfiguration.tcl
SET( MEMORYCHECK_COMMAND ${CTEST_MEMORYCHECK_COMMAND} CACHE STRING "" FORCE)
SET( MEMORYCHECK_COMMAND_OPTIONS ${CTEST_MEMORYCHECK_COMMAND_OPTIONS} CACHE STRING "" FORCE)

# Developers: please consider using these tests to verify your code!
# to obtain the necessary data files, please check the corresponding
# README files in the example folders and/or contact the EUTelescope
# software coordinators.
# !RELEASE: the data-driven tests require DESY-AFS access and are not
# useful for typical users (comment out the following lines)

# for GBL release: legacy status for tests, commented out
#  INCLUDE(jobsub/examples/datura-noDUT/testing.cmake)
#  INCLUDE(jobsub/examples/datura-alone/testing.cmake)
#  INCLUDE(jobsub/examples/anemone-2FEI4/testing.cmake)
#  INCLUDE(jobsub/examples/aconite-4chip/testing.cmake)
#  INCLUDE(jobsub/examples/aconite-4chipLocal/testing.cmake)

configure_file( ./CTestCustom.cmake.in   # set custom settings (e.g. tests to be skipped for valgrind)
	       	${PROJECT_BINARY_DIR}/CTestCustom.cmake
	      )
ELSE()
    MESSAGE( "EUTELESCOPE environment variable not defined, cannot run tests \
    	     - please run 'source build_env.sh' in the EUTel root directory first." )
ENDIF()
