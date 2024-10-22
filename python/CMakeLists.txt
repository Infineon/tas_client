# Check if python wrapper should be built
if (NOT TAS_CLIENT_API_BUILD_PYTHON)
    return()
endif()

# -----------------------------------------------------------------------------
# Python project definition
# -----------------------------------------------------------------------------
set(PYTHON_PROJECT PyTAS)
message(STATUS "Python project: ${PYTHON_PROJECT}")
set(PYTHON_PROJECT_DIR ${PROJECT_BINARY_DIR}/python/src/${PYTHON_PROJECT})
message(STATUS "Python project build path: ${PYTHON_PROJECT_DIR}")

# -----------------------------------------------------------------------------
# Find relevant dependencies
# -----------------------------------------------------------------------------
# Uncomment and spcify the root of Python installation if specific version should be used, otherwise the system default 
# is selected 
if (WIN32)
    #set(Python3_ROOT_DIR "C:\\Program Files\\Python311")
elseif (UNIX)
    #set(Python3_ROOT_DIR "/usr/bin/python3.11")
endif()

set(PYBIND11_FINDPYTHON ON)
find_package(pybind11 CONFIG REQUIRED)

if (NOT pybind11_FOUND)
    message(FATAL_ERROR "pybind11 not found ${pybind11_FOUND}. Dependecies managed by conan, run 'conan install . -o python=True' first")
endif()

# ================================ General configuration ======================================
# Solution for an open issue due to installed python debug libs: https://github.com/pybind/pybind11/issues/3403
set_target_properties(Python::Module PROPERTIES
        MAP_IMPORTED_CONFIG_DEBUG ";RELEASE")

# Setup virtual pyton environment for testing and stub gen
# Setup variables for targeting a virtual environment
set(VENV_DIR "${CMAKE_CURRENT_BINARY_DIR}/pytas_venv")
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(VENV_BIN "${VENV_DIR}/Scripts")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin" OR CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(VENV_BIN "${VENV_DIR}/bin")
else()
    message(FATAL_ERROR "Unsupported platform: ${CMAKE_SYSTEM_NAME}")
endif()

# Create virtual environment
if(NOT EXISTS ${VENV_DIR})
    message(STATUS "Creating virtual environment...")
    execute_process(
        COMMAND ${Python_EXECUTABLE} -m venv ${VENV_DIR}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
endif()

cmake_path(GET Python_EXECUTABLE FILENAME Python_EXECUTABLE_NAME)
find_program(Python_VENV_EXECUTABLE NAMES ${Python_EXECUTABLE_NAME} PATHS ${VENV_BIN} NO_CACHE NO_DEFAULT_PATH)
message(STATUS "Python executable in venv: ${Python_VENV_EXECUTABLE}")

if(NOT Python_VENV_EXECUTABLE)
    message(FATAL_ERROR "Python executable not found in virtual environment: ${Python_VENV_EXECUTABLE}")
endif()

# Install required Python packages
message(STATUS "Venv exe: ${Python_VENV_EXECUTABLE}")
execute_process(
    COMMAND ${Python_VENV_EXECUTABLE} -m pip install -r ${CMAKE_CURRENT_SOURCE_DIR}/requirements.txt
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

# =============================== CMake target - bindings_library =============================
pybind11_add_module(PyTAS
        binding/tas_python_binding.cpp
        binding/tas_python_binding.hpp
        binding/tas_python_client_server_con.cpp
        binding/tas_python_client_server_con.hpp
        binding/tas_python_client_rw_base.cpp
        binding/tas_python_client_rw_base.hpp
        binding/tas_python_client_rw.hpp
        binding/tas_python_client_chl.cpp
        binding/tas_python_client_chl.hpp
        binding/tas_python_client_trc.cpp
        binding/tas_python_client_trc.hpp
)

target_include_directories(PyTAS 
    PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/binding"
)



# -----------------------------------------------------------------------------
# Python package
# -----------------------------------------------------------------------------
file(GENERATE OUTPUT ${PYTHON_PROJECT_DIR}/__init__.py CONTENT "__version__ = \"${PROJECT_VERSION}\"\n\nfrom PyTAS import *")

# setup.py.in contains cmake variable e.g. @PYTHON_PROJECT@ and
# generator expression e.g. $<TARGET_FILE_NAME:pyFoo>
configure_file(
    ${PROJECT_SOURCE_DIR}/cmake/templates/setup.py.in
    ${PROJECT_BINARY_DIR}/python/setup.py.in
    @ONLY
)

# This step is required to trigger the generator expressions
file(GENERATE
    OUTPUT ${PROJECT_BINARY_DIR}/python/setup.py
    INPUT ${PROJECT_BINARY_DIR}/python/setup.py.in
)

add_custom_command(
    OUTPUT dist/timestamp
    COMMAND ${CMAKE_COMMAND} -E remove_directory dist
    COMMAND ${CMAKE_COMMAND} -E make_directory ${PYTHON_PROJECT_DIR}/.libs
   
    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:PyTAS> ${PYTHON_PROJECT_DIR}

    COMMAND ${Python_VENV_EXECUTABLE} setup.py sdist bdist_wheel
    COMMAND ${CMAKE_COMMAND} -E touch ${CMAKE_CURRENT_BINARY_DIR}/dist/timestamp

    DEPENDS
        setup.py

    BYPRODUCTS
        ${PYTHON_PROJECT}
        ${PYTHON_PROJECT}.egg-info
        build
        dist

    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMAND_EXPAND_LISTS
)

add_custom_target(python_package ALL
    DEPENDS
        dist/timestamp
    WORKING_DIRECTORY ${PROJECT_BINARY_DIR}/python
)

# -----------------------------------------------------------------------------
# Python tests
# -----------------------------------------------------------------------------
# Check if the Python executable exists
if(NOT Python_VENV_EXECUTABLE)
    message(FATAL_ERROR "Python executable not found in virtual environment.")
endif()

# Install the .whl file using a custom script to ensure platform independence
file(WRITE ${CMAKE_BINARY_DIR}/install_whl.py
"import os
import subprocess
import glob

whl_dir = os.path.join('${CMAKE_CURRENT_BINARY_DIR}', 'dist')
whl_files = glob.glob(os.path.join(whl_dir, '*.whl'))

if whl_files:
    for whl_file in whl_files:
        print(f'Installing {whl_file}')
        subprocess.check_call(['${Python_VENV_EXECUTABLE}', '-m', 'pip', 'install', '--force-reinstall', '--no-deps', '--upgrade', whl_file])
else:
    raise FileNotFoundError('No .whl file found in the specified directory')
")

add_custom_command(TARGET python_package POST_BUILD
    COMMAND ${Python_VENV_EXECUTABLE} ${CMAKE_BINARY_DIR}/install_whl.py
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Installing ${PYTHON_PROJECT}"
    VERBATIM
)

# add_python_test()
# CMake function to generate and build python test.
# Parameters:
#  the python filename
# e.g.:
# add_python_test(foo.py)
function(add_python_test FILE_NAME)
  message(STATUS "Configuring test ${FILE_NAME} ...")
  get_filename_component(EXAMPLE_NAME ${FILE_NAME} NAME_WE)

  if(BUILD_TESTING)
    add_test(
      NAME python_test_${EXAMPLE_NAME}
      COMMAND ${Python_VENV_EXECUTABLE} ${FILE_NAME}
      WORKING_DIRECTORY ${VENV_DIR})
  endif()
  message(STATUS "Configuring test ${FILE_NAME} done")
endfunction()

add_python_test(${CMAKE_CURRENT_SOURCE_DIR}/tests/tas_python_client_rw_test.py)
add_python_test(${CMAKE_CURRENT_SOURCE_DIR}/tests/tas_python_client_server_test.py)