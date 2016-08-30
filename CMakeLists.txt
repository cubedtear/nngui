cmake_minimum_required (VERSION 2.8.3)

project("PicoGUI")

if (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(CMAKE_CXX_FLAGS "-std=c++11")
endif()

option(PICOGUI_SDL_ENABLED "Build PicoGUI with SDL?" ON)

if (PICOGUI_SDL_ENABLED)
    add_definitions( -DPICOGUI_SDL )
    option(PICOGUI_BUILD_EXAMPLESDL  "Build PicoGUI examples with SDL2?" ON)
else()
    option(PICOGUI_BUILD_EXAMPLE "Build PicoGUI examples with openGL?" ON)
endif()

# Required libraries for linking against picogui (all targets)
set(PICOGUI_EXTRA_LIBS "")

# Platform-dependent files for libpicogui
set(PICOGUI_EXTRA_SOURCE "")

include(CheckCXXCompilerFlag)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()
string(TOUPPER "${CMAKE_BUILD_TYPE}" U_CMAKE_BUILD_TYPE)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  # Quench annoying deprecation warnings when compiling GLFW on OSX
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-deprecated-declarations")
endif()

if (MSVC)
  # Disable annoying MSVC warnings (all targets)
  add_definitions(/D "_CRT_SECURE_NO_WARNINGS")

  # Parallel build on MSVC (all targets)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")

  if (NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /arch:SSE2")

    # Disable Eigen vectorization for Windows 32 bit builds (issues with unaligned access segfaults)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /DEIGEN_DONT_ALIGN")
  endif()
endif()

# Compile with compiler warnings turned on
if(MSVC)
  if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
    string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
  endif()
elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  # Enable C++11 mode on GCC / Clang
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif()

if(PICOGUI_BUILD_EXAMPLESDL)
  set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})

  find_package(SDL2 REQUIRED)

  list(APPEND PICOGUI_EXTRA_LIBS ${SDL2_LIBRARY})
endif()

# Required core libraries on various platforms
if (WIN32) 
  list(APPEND PICOGUI_EXTRA_LIBS opengl32)
elseif (APPLE)
  find_library(cocoa_library Cocoa)
  find_library(opengl_library OpenGL)
  find_library(corevideo_library CoreVideo)
  find_library(iokit_library IOKit)
  list(APPEND PICOGUI_EXTRA_LIBS ${cocoa_library} ${opengl_library} ${corevideo_library} ${iokit_library})
  list(APPEND PICOGUI_EXTRA_SOURCE src/darwin.mm)
elseif(CMAKE_SYSTEM MATCHES "Linux")
  list(APPEND PICOGUI_EXTRA_LIBS GL Xxf86vm Xrandr Xinerama Xcursor Xi X11 pthread dl rt)
endif()

include_directories(ext src ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_BINARY_DIR})

# Needed to generated files
include_directories(${CMAKE_CURRENT_BINARY_DIR})

message(${PICOGUI_EXTRA_LIBS})

set(PICOGUI_BASIC_SOURCE "")
list(APPEND PICOGUI_BASIC_SOURCE
     src/picogui.cpp
     src/picogui_resources.cpp
     ext/nanovg/nanovg.c)

# Quench warnings while compiling NanoVG
if (CMAKE_COMPILER_IS_GNUCC)
  set_source_files_properties(ext/nanovg/src/nanovg.c PROPERTIES COMPILE_FLAGS -Wno-unused-result)
elseif(MSVC)
  set_source_files_properties(ext/nanovg/src/nanovg.c PROPERTIES COMPILE_FLAGS "/wd4005 /wd4456 /wd4457")
endif()

# Build example application if desired
if(PICOGUI_BUILD_EXAMPLESDL)
  add_executable(example1 ${PICOGUI_EXTRA_SOURCE} ${PICOGUI_BASIC_SOURCE} examples/example1.cpp )
  add_executable(example2 ${PICOGUI_EXTRA_SOURCE} ${PICOGUI_BASIC_SOURCE} examples/example2.cpp )

  target_link_libraries(example1 ${PICOGUI_EXTRA_LIBS})
  target_link_libraries(example2 ${PICOGUI_EXTRA_LIBS})

  # Copy icons for example application
  file(COPY resources/icons DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
endif()
