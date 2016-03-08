project(area)

cmake_minimum_required(VERSION 2.8)

# Turn compiler warnings up to 11, at least with gcc.  
if (CMAKE_BUILD_TOOL MATCHES "make")
    MESSAGE(STATUS "setting gcc options: -Wall -Werror -Wno-deprecated -pedantic-errors")
    # NON-optimized build:
    # add_definitions( -Wall  -Wno-deprecated -Werror -pedantic-errors)
    add_definitions(-fPIC)
endif (CMAKE_BUILD_TOOL MATCHES "make")

option(BUILD_TYPE
  "Build type: Release=ON/Debug=OFF  " ON)

if (BUILD_TYPE)
    MESSAGE(STATUS " CMAKE_BUILD_TYPE = Release")
    set(CMAKE_BUILD_TYPE Release)
endif(BUILD_TYPE)

if (NOT BUILD_TYPE)
    MESSAGE(STATUS " CMAKE_BUILD_TYPE = Debug")
    set(CMAKE_BUILD_TYPE Debug)
endif(NOT BUILD_TYPE)

option(PYTHON_DIR "Python executional dir")

# -------------- start python search code --------------
if ("${PYTHON_DIR}" STREQUAL "OFF")
  # we havent set any specific python, use system defaults
  set(PYTHON_DIR "")

else("${PYTHON_DIR}" STREQUAL "OFF")
  # we have set a specific python install, correct accordingly
  # such as macports for example
  set(PYTHON_DIR "${PYTHON_DIR}/")

  # this figures out the Python include directories and adds them to the
  # header file search path so that find_package(PythonLibs) searches in the correct place
  execute_process(
    COMMAND ${PYTHON_DIR}python-config --includes OUTPUT_VARIABLE Python_Includes
  )
  string(REGEX REPLACE "-I([^;\n ]+).*" "\\1" Python_Includes ${Python_Includes})
  MESSAGE(STATUS " PYTHON_INCLUDES = ${Python_Includes}")

  # need to set these when we have multipe pythons installed
  set(PYTHON_INCLUDE_DIR ${Python_Includes})
  execute_process(
    COMMAND ${PYTHON_DIR}python-config --prefix OUTPUT_VARIABLE Python_Prefix
  )
  string(REGEX REPLACE "([^;\n ]+).*" "\\1" Python_Prefix ${Python_Prefix})
  MESSAGE(STATUS " PYTHON_prefix = ${Python_Prefix}")

  # find the library name of selected python install
  find_library(PYTHON_LIBRARY NAMES python python2.7 python2.6 python3.0
                 python3.1 python3.2 python 3.3 python3.4 python3.5
               PATHS "${Python_Prefix}/lib/" NO_DEFAULT_PATH NO_SYSTEM_ENVIRONMENT_PATH)

  MESSAGE(STATUS " PYTHON_LIBRARY = ${PYTHON_LIBRARY}")

  if (PYTHON_LIBRARY MATCHES PYTHON_LIBRARY-NOTFOUND)
    MESSAGE(FATAL_ERROR "*** Python lib not found!")
  endif()

endif("${PYTHON_DIR}" STREQUAL "OFF")

find_package ( PythonLibs REQUIRED)

message(STATUS "PYTHON_LIBRARY:FILEPATH=${PYTHON_LIBRARY}")
message(STATUS "PYTHON_LIBRARIES:FILEPATH=${PYTHON_LIBRARIES}")
message(STATUS "PYTHON_INCLUDE_DIR:FILEPATH=${PYTHON_INCLUDE_DIR}")
message(STATUS "PYTHON_FRAMEWORK_INCLUDES=${PYTHON_FRAMEWORK_INCLUDES}")
message(STATUS "PYTHONLIBS_VERSION_STRING=${PYTHONLIBS_VERSION_STRING}")
message(STATUS "Python_FRAMEWORKS=${Python_FRAMEWORKS}")

get_filename_component(PYTHON_LIBRARY_DIR ${PYTHON_LIBRARY} DIRECTORY)
message(STATUS "PYTHON_LIBRARY_DIR:FILEPATH=${PYTHON_LIBRARY_DIR}")

include_directories(${PYTHON_INCLUDE_DIR})
link_directories(${PYTHON_LIBRARY_DIR})

# ----------------------- END Python search code ---------------------

find_package( Boost COMPONENTS python REQUIRED)  # find BOOST and boost-python
if(Boost_FOUND)
    include_directories(${Boost_INCLUDE_DIRS})
    MESSAGE(STATUS "found Boost: " ${Boost_LIB_VERSION})
    MESSAGE(STATUS "boost-incude dirs are: " ${Boost_INCLUDE_DIRS})
    MESSAGE(STATUS "boost-python lib is: " ${Boost_PYTHON_LIBRARY})
    MESSAGE(STATUS "boost_LIBRARY_DIRS is: " ${Boost_LIBRARY_DIRS})
    MESSAGE(STATUS "Boost_LIBRARIES is: " ${Boost_LIBRARIES})    
endif()


include_directories(${CMAKE_CURRENT_BINARY_DIR})

# this defines the source-files
set(AREA_SRC
    ${area_SOURCE_DIR}/PythonStuff.cpp

    ${area_SOURCE_DIR}/Arc.cpp
    ${area_SOURCE_DIR}/Area.cpp
    ${area_SOURCE_DIR}/AreaBoolean.cpp
    ${area_SOURCE_DIR}/AreaDxf.cpp
    ${area_SOURCE_DIR}/AreaOrderer.cpp
    ${area_SOURCE_DIR}/AreaPocket.cpp
    ${area_SOURCE_DIR}/Circle.cpp
    ${area_SOURCE_DIR}/Curve.cpp
    ${area_SOURCE_DIR}/dxf.cpp
    
    ${area_SOURCE_DIR}/kbool/src/booleng.cpp
    ${area_SOURCE_DIR}/kbool/src/record.cpp
    ${area_SOURCE_DIR}/kbool/src/scanbeam.cpp
    ${area_SOURCE_DIR}/kbool/src/graph.cpp
    ${area_SOURCE_DIR}/kbool/src/graphlst.cpp
    ${area_SOURCE_DIR}/kbool/src/instonly.cpp
    ${area_SOURCE_DIR}/kbool/src/line.cpp
    ${area_SOURCE_DIR}/kbool/src/link.cpp
    ${area_SOURCE_DIR}/kbool/src/lpoint.cpp    
    ${area_SOURCE_DIR}/kbool/src/node.cpp
     
    ${area_SOURCE_DIR}/kurve/Construction.cpp
    ${area_SOURCE_DIR}/kurve/Finite.cpp
    ${area_SOURCE_DIR}/kurve/kurve.cpp
    ${area_SOURCE_DIR}/kurve/Matrix.cpp
    ${area_SOURCE_DIR}/kurve/offset.cpp
)

# this defines the headers
if(DEFINED INCLUDE_INSTALL_DIR)
  set(includedir ${INCLUDE_INSTALL_DIR})
else(DEFINED INCLUDE_INSTALL_DIR)
  set(INCLUDE_INSTALL_DIR include)
  set(includedir ${CMAKE_INSTALL_PREFIX}/${INCLUDE_INSTALL_DIR})
endif(DEFINED INCLUDE_INSTALL_DIR)

file(GLOB headers "${CMAKE_CURRENT_SOURCE_DIR}/*.h")
install(FILES ${headers} DESTINATION ${INCLUDE_INSTALL_DIR}/area COMPONENT headers)

# include directories
include_directories( 
    ${area_SOURCE_DIR} 
    ${area_SOURCE_DIR}/kurve/
    ${area_SOURCE_DIR}/kbool/src/
)

# this makes the Python module
add_library(
    area
    MODULE
    ${AREA_SRC}
)
target_link_libraries(area ${Boost_LIBRARIES} ${PYTHON_LIBRARY})
set_target_properties(area PROPERTIES PREFIX "") 

# this figures out where to install the Python modules
execute_process(
    COMMAND "${PYTHON_DIR}python" -c "from distutils.sysconfig import get_python_lib; print get_python_lib()"
    OUTPUT_VARIABLE Python_site_packages
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
MESSAGE(STATUS "${PYTHON_DIR}python")
MESSAGE(STATUS "Python libraries will be installed to: " ${Python_site_packages})

# this installs the python library
install(
	TARGETS area
	LIBRARY DESTINATION ${Python_site_packages}
)
