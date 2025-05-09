cmake_minimum_required(VERSION 3.20.0)
project(csp-example-struct VERSION "0.0.1")
set(CMAKE_CXX_STANDARD 17)

include(CheckCCompilerFlag)

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(MACOS ON)
    set(LINUX OFF)
else()
    set(MACOS OFF)
    set(LINUX ON)
endif()

list(PREPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/../../../cpp/cmake/modules")
set(ENV{PYTHONPATH} "${CMAKE_SOURCE_DIR}/../../../:$ENV{PYTHONPATH}")

set(CMAKE_MACOSX_RPATH TRUE)
set(CMAKE_SKIP_RPATH FALSE)
set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
set(CMAKE_INSTALL_NAME_DIR "@rpath")
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(NOT DEFINED PYTHON_VERSION)
    set(PYTHON_VERSION 3.11)
endif()

find_package(Color)

if(MACOS)
    # fix for threads on osx
    # assume built-in pthreads on MacOS
    set(CMAKE_THREAD_LIBS_INIT "-lpthread")
    set(CMAKE_HAVE_THREADS_LIBRARY 1)
    set(CMAKE_USE_WIN32_THREADS_INIT 0)
    set(CMAKE_USE_PTHREADS_INIT 1)
    set(THREADS_PREFER_PTHREAD_FLAG ON)

    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.13" CACHE STRING "Minimum OS X deployment version")

    # don't link against build python
    # https://blog.tim-smith.us/2015/09/python-extension-modules-os-x/
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -undefined dynamic_lookup")

    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-ld_classic")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-ld_classic")

    # Support cross build
    check_c_compiler_flag("-arch x86_64" x86_64Supported)
    check_c_compiler_flag("-arch arm64" arm64Supported)

    if(x86_64Supported AND arm64Supported)
        set(CMAKE_OSX_ARCHITECTURES "x86_64;arm64" CACHE STRING "Build universal architecture for OSX" FORCE)
    elseif(x86_64Supported)
        set(CMAKE_REQUIRED_LINK_OPTIONS "-arch;x86_64")
        set(CMAKE_OSX_ARCHITECTURES "x86_64" CACHE STRING "Build universal architecture for OSX" FORCE)
    elseif(arm64Supported)
        set(CMAKE_REQUIRED_LINK_OPTIONS "-arch;arm64")
        set(CMAKE_OSX_ARCHITECTURES "arm64" CACHE STRING "Build universal architecture for OSX" FORCE)
    endif()

    set(CMAKE_INSTALL_RPATH "@loader_path/../csp/lib")

    message("${Cyan}Use python shared libraries${ColorReset}")
    find_package(Python ${PYTHON_VERSION} EXACT REQUIRED COMPONENTS Interpreter Development)

    # Run with exact version so its cached for pybind
    find_package(PythonInterp ${PYTHON_VERSION} EXACT REQUIRED)
    find_package(PythonLibs ${PYTHON_VERSION} EXACT REQUIRED)

    link_directories(${Python_LIBRARY_DIRS})
elseif(LINUX)
    set(CMAKE_INSTALL_RPATH "\$ORIGIN/../csp/lib")

    # Manylinux docker images have no shared libraries
    # The instead use a statically built python.
    # Cmake's default FindPython can't find the python headers
    # without also finding (or failing to find) the python libraries
    # so we use a custom FindPythonHeaders that is the same as the
    # default, but ignores when the python libraries can't be found.
    message("${Red}Manylinux build has no python shared libraries${ColorReset}")
    find_package(Python ${PYTHON_VERSION} EXACT REQUIRED COMPONENTS Interpreter)
    find_package(PythonHeaders ${PYTHON_VERSION} EXACT REQUIRED)

    # Run with exact version so its cached for pybind
    find_package(PythonInterp ${PYTHON_VERSION} EXACT REQUIRED)
endif()

message("${Cyan}Using Python ${Python_VERSION}\nPython_INCLUDE_DIRS: ${Python_INCLUDE_DIRS}\nPython_LIBRARIES: ${Python_LIBRARIES}\nPython_EXECUTABLE: ${Python_EXECUTABLE} ${ColorReset}")
include_directories(${Python_INCLUDE_DIRS})

# prefix is _ by default
set(CMAKE_SHARED_LIBRARY_PREFIX _)

# shared suffix is .so for both linux and mac
set(CMAKE_SHARED_LIBRARY_SUFFIX .so)

find_package(CSP REQUIRED)
message("${Cyan}Found CSP:\n\tincludes in: ${CSP_INCLUDE_DIR}\n\tlibraries in: ${CSP_LIBS_DIR}${ColorReset}")
include_directories(${CSP_INCLUDE_DIR})

find_package(csp_autogen REQUIRED)
csp_autogen(mystruct.struct mystruct STRUCT_AUTOGEN_HEADER STRUCT_AUTOGEN_SOURCE)

add_library(mystruct SHARED struct.cpp ${STRUCT_AUTOGEN_SOURCE})
target_link_libraries(mystruct ${CSP_LIBRARY})
target_include_directories(mystruct PRIVATE "${CMAKE_BINARY_DIR}/csp_autogen")
set_target_properties(mystruct PROPERTIES PUBLIC_HEADER "${STRUCT_AUTOGEN_HEADER}")

install(TARGETS mystruct LIBRARY DESTINATION mystruct PUBLIC_HEADER DESTINATION mystruct/include)
