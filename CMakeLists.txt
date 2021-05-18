##################################################    Project     ##################################################
cmake_minimum_required(VERSION 3.10 FATAL_ERROR)
project               (noise VERSION 1.0 LANGUAGES CXX)
list                  (APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
set_property          (GLOBAL PROPERTY USE_FOLDERS ON)
set                   (CMAKE_CXX_STANDARD 17)
set                   (CMAKE_CXX_VISIBILITY_PRESET hidden)
set                   (CMAKE_VISIBILITY_INLINES_HIDDEN 1)

include               (set_max_warning_level)
set_max_warning_level ()

##################################################    Options     ##################################################
option(BUILD_SHARED_LIBS "Build shared (dynamic) libraries." OFF)
option(BUILD_TESTS "Build tests." OFF)
option(BUILD_DOCUMENTATION "Create doxygen documentation for developers" OFF)
option(BUILD_UTILS "Build utility functions for use with libnoise" ON)
option(BUILD_EXAMPLES "Build libnoise examples" ON)

##################################################    Sources     ##################################################
file(GLOB_RECURSE PROJECT_HEADERS include/*.h include/*.hpp)
file(GLOB_RECURSE PROJECT_SOURCES source/*.c source/*.cpp)
file(GLOB_RECURSE PROJECT_CMAKE_UTILS cmake/*.cmake)
file(GLOB_RECURSE PROJECT_MISC *.md *.txt)
set (PROJECT_FILES 
  ${PROJECT_HEADERS} 
  ${PROJECT_SOURCES} 
  ${PROJECT_CMAKE_UTILS} 
  ${PROJECT_MISC})

include            (assign_source_group)
assign_source_group(${PROJECT_FILES})

##################################################  Dependencies  ##################################################
include(import_library)

# ADD LIBRARIES HERE.
# Examples:
# - Header Only:
#   find_package  (GLM REQUIRED)
#   import_library(GLM_INCLUDE_DIRS)
# - Identical Debug and Release:
#   find_package  (OpenGL REQUIRED)
#   import_library(OPENGL_INCLUDE_DIR OPENGL_LIBRARIES)
# - Separate Debug and Release:
#   find_package  (Boost REQUIRED iostreams)
#   import_library(Boost_INCLUDE_DIRS Boost_IOSTREAMS_LIBRARY_DEBUG Boost_IOSTREAMS_LIBRARY_RELEASE)
# - You may also set the PROJECT_INCLUDE_DIRS and PROJECT_LIBRARIES instead of using import_library.

##################################################    Targets     ##################################################
add_library(${PROJECT_NAME} ${PROJECT_FILES})
target_include_directories(${PROJECT_NAME} PUBLIC 
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<INSTALL_INTERFACE:include> PRIVATE source)
target_include_directories(${PROJECT_NAME} PUBLIC ${PROJECT_INCLUDE_DIRS})
target_link_libraries     (${PROJECT_NAME} PUBLIC ${PROJECT_LIBRARIES})
target_compile_definitions(${PROJECT_NAME} PUBLIC ${PROJECT_COMPILE_DEFINITIONS})
set_target_properties     (${PROJECT_NAME} PROPERTIES LINKER_LANGUAGE CXX)
set_target_properties     (${PROJECT_NAME} PROPERTIES OUTPUT_NAME "noise" )

string (TOUPPER ${PROJECT_NAME} PROJECT_NAME_UPPER)
if(NOT BUILD_SHARED_LIBS)
  set_target_properties     (${PROJECT_NAME} PROPERTIES COMPILE_FLAGS -D${PROJECT_NAME_UPPER}_STATIC)
  target_compile_definitions(${PROJECT_NAME} PUBLIC NOISE_STATIC)
else()
  target_compile_definitions(${PROJECT_NAME} PRIVATE NOISE_BUILD_DLL)
endif()

if (BUILD_UTILS)
	add_subdirectory(noiseutils)
endif()

if (BUILD_EXAMPLES)
	add_subdirectory(examples)
endif()

if (BUILD_DOCUMENTATION)
	add_subdirectory(doc)
endif()


##################################################    Testing     ##################################################
if(BUILD_TESTS)
  enable_testing     ()
  set                (TEST_MAIN_NAME test_main)
  set                (TEST_MAIN_SOURCES tests/internal/main.cpp)
  add_library        (${TEST_MAIN_NAME} OBJECT ${TEST_MAIN_SOURCES})
  set_property       (TARGET ${TEST_MAIN_NAME} PROPERTY FOLDER tests)
  assign_source_group(${TEST_MAIN_SOURCES})

  file(GLOB PROJECT_TEST_CPPS tests/*.cpp)
  foreach(_SOURCE ${PROJECT_TEST_CPPS})
    get_filename_component(_NAME ${_SOURCE} NAME_WE)
    add_executable        (${_NAME} ${_SOURCE} $<TARGET_OBJECTS:${TEST_MAIN_NAME}>)
    target_link_libraries (${_NAME} ${PROJECT_NAME})
    add_test              (${_NAME} ${_NAME})
    set_property          (TARGET ${_NAME} PROPERTY FOLDER tests)
    assign_source_group   (${_SOURCE})
  endforeach()
endif()

##################################################  Installation  ##################################################
install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}-config
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin)
install(DIRECTORY include/ DESTINATION include)
install(EXPORT  ${PROJECT_NAME}-config DESTINATION cmake)
export (TARGETS ${PROJECT_NAME}        FILE        ${PROJECT_NAME}-config.cmake)
