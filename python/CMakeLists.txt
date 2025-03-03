pybind11_add_module(manifpy MODULE
  bindings_rn.cpp
  bindings_so2.cpp
  bindings_so3.cpp
  bindings_se2.cpp
  bindings_se3.cpp
  bindings_se_2_3.cpp
  bindings_manif.cpp
)
target_link_libraries(manifpy PRIVATE ${PROJECT_NAME})
# Eigen and numpy have different default storage order.
# See e.g. https://pybind11.readthedocs.io/en/stable/advanced/cast/eigen.html#storage-orders
target_compile_definitions(manifpy PRIVATE EIGEN_DEFAULT_TO_ROW_MAJOR)

# Specify that C++11 features are required
target_compile_features(manifpy PRIVATE cxx_nullptr)
set_property(TARGET manifpy PROPERTY CXX_STANDARD_REQUIRED ON)
set_property(TARGET manifpy PROPERTY CXX_EXTENSIONS OFF)

if (CALL_FROM_SETUP_PY)
  # cmake-build-extension sets the full absolute path as CMAKE_INSTALL_PREFIX.
  set(MANIFPY_PKGDIR "${CMAKE_INSTALL_PREFIX}")
  set(MANIFPY_INSTDIR "${CMAKE_INSTALL_PREFIX}")
# Permit to specify MANIFPY_INSTDIR as an option
elseif(NOT DEFINED MANIFPY_PKGDIR)
  # 'distutils.sysconfig.get_python_lib' returns the absolute path of Python
  # by default a global location managed by the distro e.g. /usr/lib/python.
  #
  # pybind11 and FindPython3 set respectively PYTHON_SITE_PACKAGES/Python3_SITELIB
  # from 'distutils.sysconfig.get_python_lib'
  #
  # Those are especially annoying on Ubuntu since it has
  # some hardcoded paths in python3.x/site.py
  #
  # `sysconfig.get_path` may return paths that does not even exists.
  #
  # So below we retrieve the first site-package path from 'site.getsitepackages()'.

  execute_process(
    COMMAND
    ${PYTHON_EXECUTABLE} -c "import site; print(site.getsitepackages()[0])"
    OUTPUT_VARIABLE _PYTHON_SITE_PACKAGE OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  set(MANIFPY_PKGDIR "${_PYTHON_SITE_PACKAGE}")
  set(MANIFPY_INSTDIR "${MANIFPY_PKGDIR}/manifpy")
else()
  set(MANIFPY_INSTDIR "${MANIFPY_PKGDIR}/manifpy")
endif()

message(STATUS "Installing manifpy in ${MANIFPY_INSTDIR}")

# Setup installation path
install(TARGETS manifpy COMPONENT python DESTINATION "${MANIFPY_INSTDIR}")

# Create the Python package in the build tree for testing purposes
set(MANIFPY_BUILDDIR "${CMAKE_BINARY_DIR}/manifpy")
set_target_properties(
  manifpy PROPERTIES
  OUTPUT_NAME _bindings
  LIBRARY_OUTPUT_DIRECTORY "${MANIFPY_BUILDDIR}")

# Create the __init__.py file
file(
  GENERATE
  OUTPUT "${MANIFPY_BUILDDIR}/__init__.py"
  CONTENT "from manifpy._bindings import *\n")

# Install the __init__.py file
install(
  FILES "${MANIFPY_BUILDDIR}/__init__.py"
  DESTINATION ${MANIFPY_INSTDIR})
  
# Install pip metadata files to ensure that manif installed via CMake is listed by pip list
# See https://packaging.python.org/specifications/recording-installed-packages/
# and https://packaging.python.org/en/latest/specifications/core-metadata/#core-metadata
if (NOT CALL_FROM_SETUP_PY)
  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/METADATA "")
  file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/METADATA "Metadata-Version: 2.1\n")
  file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/METADATA "Name: manifpy\n")
  file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/METADATA "Version: ${PROJECT_VERSION}\n")
  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/INSTALLER "cmake\n")
  install(
    FILES "${CMAKE_CURRENT_BINARY_DIR}/METADATA" "${CMAKE_CURRENT_BINARY_DIR}/INSTALLER"
    DESTINATION ${MANIFPY_PKGDIR}/manifpy-${PROJECT_VERSION}.dist-info)
endif()
