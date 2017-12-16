CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

INCLUDE(cmake/base.cmake)
INCLUDE(cmake/boost.cmake)
INCLUDE(cmake/eigen.cmake)
INCLUDE(cmake/python.cmake)

SET(PROJECT_NAME foo)
SET(PROJECT_DESCRIPTION "test project")
SET(PROJECT_URL "http://github.com/foo")
OPTION(INSTALL_DOCUMENTATION "Generate and install the documentation" FALSE)

# Disable -Werror on Unix for now.
SET(CXX_DISABLE_WERROR True)
SET(CMAKE_VERBOSE_MAKEFILE True)

SETUP_PROJECT()
STRING(REPLACE "-pedantic" "" CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})

IF(APPLE)
  SET(CMAKE_MACOSX_RPATH TRUE)
  SET(CMAKE_SKIP_BUILD_RPATH FALSE)
  SET(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
  SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

  #SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
  LIST(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
  IF("${isSystemDir}" STREQUAL "-1")
     SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
  ENDIF("${isSystemDir}" STREQUAL "-1")
ENDIF(APPLE)

# ----------------------------------------------------
# --- OPTIONS  ---------------------------------------
# ----------------------------------------------------
OPTION (EIGEN_NUMPY_ALIGNED "Directly aligned data between Numpy and Eigen" OFF)
OPTION (BUILD_UNIT_TESTS "Build the unitary tests" OFF)

IF(EIGEN_NUMPY_ALIGNED)
  ADD_DEFINITIONS(-DEIGENPY_ALIGNED)
ENDIF(EIGEN_NUMPY_ALIGNED)

# ----------------------------------------------------
# --- DEPENDANCIES -----------------------------------
# ----------------------------------------------------
ADD_REQUIRED_DEPENDENCY("eigen3 >= 3.0.5")

# Spetcial care of Eigen3 greater than 3.2.10
IF(EIGEN3_FOUND)
  IF(${EIGEN3_VERSION} VERSION_GREATER "3.2.10")
    ADD_DEFINITIONS(-DEIGEN3_FUTURE)
    SET(EIGEN3_FUTURE TRUE)
    PKG_CONFIG_APPEND_CFLAGS("-DEIGEN3_FUTURE")
    IF(${EIGEN3_VERSION} VERSION_LESS "3.3.0")
      ADD_DEFINITIONS(-DEIGEN3_BETA_3_2_9x)
      SET(EIGEN3_FUTURE TRUE)
      PKG_CONFIG_APPEND_CFLAGS("-DEIGEN3_BETA_3_2_9x")
    ENDIF(${EIGEN3_VERSION} VERSION_LESS "3.3.0")
  ELSE(${EIGEN3_VERSION} VERSION_GREATER "3.2.10")
    SET(EIGEN3_FUTURE FALSE)
  ENDIF(${EIGEN3_VERSION} VERSION_GREATER "3.2.10")
ENDIF(EIGEN3_FOUND)

SET(BOOST_COMPONENTS python)
SEARCH_FOR_BOOST()
# Add Boost path to include directories.
INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIRS})

FINDPYTHON(2.7 EXACT REQUIRED)
INCLUDE_DIRECTORIES(${PYTHON_INCLUDE_DIRS})
FIND_NUMPY()
INCLUDE_DIRECTORIES(${NUMPY_INCLUDE_DIRS})

# check for dependency using pkg-config
SET(PKG_CONFIG_PYWRAP_REQUIRES "eigenpy >= 1.3.1" "pinocchio >= 0.0.1")
FOREACH(dep ${PKG_CONFIG_PYWRAP_REQUIRES})
  ADD_COMPILE_DEPENDENCY(${dep})
ENDFOREACH(dep ${PKG_CONFIG_PYWRAP_REQUIRES})

MESSAGE(Boost_INCLUDE_DIRS: ${Boost_INCLUDE_DIRS})
MESSAGE(PYTHON_INCLUDE_DIRS: ${PYTHON_INCLUDE_DIRS})
MESSAGE(NUMPY_INCLUDE_DIRS: ${NUMPY_INCLUDE_DIRS})

# ----------------------------------------------------
# --- INCLUDE ----------------------------------------
# ----------------------------------------------------
ADD_LIBRARY(matrix SHARED matrix)
PKG_CONFIG_USE_DEPENDENCY(matrix eigen3)
PKG_CONFIG_USE_DEPENDENCY(matrix eigenpy)
PKG_CONFIG_USE_DEPENDENCY(matrix pinocchio)

TARGET_LINK_BOOST_PYTHON(matrix)
SET_TARGET_PROPERTIES(matrix PROPERTIES PREFIX "../")

IF(APPLE)
  # We need to change the extension for python bindings
  SET_TARGET_PROPERTIES(matrix PROPERTIES SUFFIX ".so")
ENDIF(APPLE)


# ----------------------------------------------------
# --- EXECUTABLES ------------------------------------
# ----------------------------------------------------

IF(EIGEN_NUMPY_ALIGNED)
  PKG_CONFIG_APPEND_CFLAGS("-DEIGENPY_ALIGNED")
ENDIF(EIGEN_NUMPY_ALIGNED)

PKG_CONFIG_APPEND_LIBS(${PROJECT_NAME})
PKG_CONFIG_APPEND_CFLAGS("-I${PYTHON_INCLUDE_DIRS}")
PKG_CONFIG_APPEND_CFLAGS("-I${NUMPY_INCLUDE_DIRS}")
PKG_CONFIG_APPEND_BOOST_LIBS(${BOOST_COMPONENTS})

SETUP_PROJECT_FINALIZE()
