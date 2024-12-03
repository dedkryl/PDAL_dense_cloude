#
# FauxPlugin CMake configuration
#

set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)

PDAL_ADD_PLUGIN(fauxplugin_kernel_libname kernel fauxplugin
    FILES
        kernel/FauxPluginKernel.cpp
)
