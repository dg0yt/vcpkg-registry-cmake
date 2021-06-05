#
# Copyright 2021 Kai Pastor <dg0yt@darc.de>
#
# License: MIT

cmake_minimum_required(VERSION 3.7 FATAL_ERROR)
if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.19")
    cmake_policy(SET CMP0114 NEW)
endif()
include(ExternalProject)
find_package(Git REQUIRED)

#
# Custom registry setup
#
set(REGISTRY_CHECKOUT "" CACHE PATH
  "Directory of the custom git registry checkout"
)
option(REGISTRY_READ_CONFIG "Read configuration from the custom registry" ON)


if(REGISTRY_CHECKOUT)
    if(NOT EXISTS "${REGISTRY_CHECKOUT}")
       message(FATAL_ERROR "The custom registry directory does not exist.")
    endif()
    set(registry_config "${REGISTRY_CHECKOUT}/registry-config.cmake")
    if(REGISTRY_READ_CONFIG AND EXISTS "${registry_config}")
        # To preset variables in a custom registry,
        # create a top-level file `registry-config.cmake`.
        # Setting cache variables allows overriding by users,
        # but will not pick updates.
        # Setting normal variables prevents overriding
        # and will always be picked
        function(load_registry_config file)
            set(vars
              REGISTRY_NAME
              VCPKG_BASELINE
              VCPKG_IMPORTS
            )
            foreach(var IN LISTS vars)
                set(${var} "NOTFOUND")
            endforeach()
            include("${file}")
            foreach(var IN LISTS vars)
                if(NOT "${${var}}" STREQUAL "NOTFOUND")
                    set(${var} "${${var}}" PARENT_SCOPE)
                endif()
            endforeach()
        endfunction()
        load_registry_config("${registry_config}")
        set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${registry_config}")
    endif()
endif()

# Unused when set by registry
set(REGISTRY_NAME "custom-registry" CACHE STRING
  "CMake project name of the custom registry project"
)
# Unused when set by registry
set(VCPKG_IMPORTS "" CACHE STRING
  "Ports to import from vcpkg, separated by semicolon"
)

option(REGISTRY_USE_AS_OVERLAY "Pass the custom registry as overlay ports to vcpkg" ON)

# Generic vcpkg options
option(VCPKG_DISABLE_METRICS  "Disable vcpkg telemetry" ON)
option(VCPKG_FORMAT_MANIFEST  "Re-format manifest when building" ON)
option(VCPKG_INSTALL_DEBUG    "Enable vcpkg debug output" OFF)
option(VCPKG_INSTALL_EDITABLE "Build packages in editable mode" ON)

set(VCPKG_DOWNLOADS_ROOT   "@BINARY_DIR@/downloads" CACHE PATH
  "Directory for storing downloaded sources and tools"
)
set(VCPKG_BUILDTREES_ROOT "@BINARY_DIR@/buildtrees" CACHE PATH
  "Directory for storing build trees"
)
set(VCPKG_PACKAGES_ROOT    "@BINARY_DIR@/packages" CACHE PATH
  "Directory for storing unpackaged packages"
)
set(VCPKG_INSTALL_ROOT     "@BINARY_DIR@/installed" CACHE PATH
  "Directory for installation roots"
)

project("${REGISTRY_NAME}" LANGUAGES NONE)


if(CMAKE_HOST_WIN32)
    set(vcpkg_bootstrap [[.\bootstrap-vcpkg.bat]])
    set(vcpkg [[.\vcpkg]])
else()
    set(vcpkg_bootstrap [[./bootstrap-vcpkg.sh]])
    set(vcpkg [[./vcpkg]])
endif()

if(VCPKG_DISABLE_METRICS)
    set(vcpkg_configure_metrics "${CMAKE_COMMAND}" -E touch "<SOURCE_DIR>/vcpkg.disable-metrics")
    list(APPEND vcpkg_bootstrap "-disableMetrics")
else()
    set(vcpkg_configure_metrics "${CMAKE_COMMAND}" -E remove -f "<SOURCE_DIR>/vcpkg.disable-metrics")
endif()	

if(VCPKG_BASELINE STREQUAL "HEAD" OR NOT DEFINED VCPKG_BASELINE)
    set(vcpkg_registry_head "master")
else()
    set(vcpkg_registry_head "${VCPKG_BASELINE}")
endif()

# vcpkg repo
set(VCPKG_ROOT "${CMAKE_CURRENT_BINARY_DIR}/vcpkg")
add_custom_command(OUTPUT "${VCPKG_ROOT}/.stamp"
  COMMAND "${CMAKE_COMMAND}" -E make_directory "${VCPKG_ROOT}"
  COMMAND "${CMAKE_COMMAND}" -E chdir "${VCPKG_ROOT}" "${GIT_EXECUTABLE}" init .
  COMMAND "${CMAKE_COMMAND}" -E touch "${VCPKG_ROOT}/.stamp"
)

configure_file("config/vcpkg-head.in" "${CMAKE_CURRENT_BINARY_DIR}/vcpkg-head" @ONLY)
add_custom_command(OUTPUT "vcpkg.stamp"
  COMMAND "${GIT_EXECUTABLE}" fetch "https://github.com/microsoft/vcpkg"
  COMMAND "${GIT_EXECUTABLE}" checkout "${vcpkg_registry_head}"
  COMMAND "${CMAKE_COMMAND}" -E touch "${CMAKE_CURRENT_BINARY_DIR}/vcpkg.stamp"
  WORKING_DIRECTORY "${VCPKG_ROOT}"
  SOURCE  "${CMAKE_CURRENT_BINARY_DIR}/vcpkg-head"
  DEPENDS "${VCPKG_ROOT}/.stamp"
)

add_custom_target(vcpkg-root
  DEPENDS "vcpkg.stamp"
)

# vcpkg tool
add_custom_command(OUTPUT "vcpkg-tool.stamp"
  COMMAND ${vcpkg_bootstrap}
  COMMAND "${CMAKE_COMMAND}" -E touch "${CMAKE_CURRENT_BINARY_DIR}/vcpkg-tool.stamp"
  WORKING_DIRECTORY "${VCPKG_ROOT}"
  DEPENDS vcpkg-root
)
add_custom_target(vcpkg-tool
  DEPENDS "vcpkg-tool.stamp"
)

# vcpkg configuration
if(REGISTRY_USE_AS_OVERLAY)
    set(OVERLAY_PORTS_OPTION "--overlay-ports=@REGISTRY_CHECKOUT@/ports")
else()
    set(OVERLAY_PORTS_OPTION "")
endif()
configure_file("config/options.in" "${CMAKE_CURRENT_BINARY_DIR}/options.in" @ONLY)

list(SORT VCPKG_IMPORTS)
list(REMOVE_DUPLICATES VCPKG_IMPORTS)
configure_file("config/update-configuration.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/update-configuration.cmake" @ONLY)
add_custom_target(vcpkg-configuration
  COMMAND "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/update-configuration.cmake"
  SOURCES
    config/options.in
    config/update-configuration.cmake.in
    config/update-version.cmake.in
    config/vcpkg-configuration.json.in
    config/vcpkg-head.in
  DEPENDS vcpkg-root
)

# vcpkg completely
string(REPLACE "@BINARY_DIR@" "${CMAKE_CURRENT_BINARY_DIR}" install_root "${VCPKG_INSTALL_ROOT}")
add_custom_target(vcpkg
  # https://github.com/microsoft/vcpkg/issues/18268
  COMMAND "${CMAKE_COMMAND}" -E remove -f "${install_root}/vcpkg/vcpkg-lock.json"
  DEPENDS vcpkg vcpkg-root vcpkg-tool vcpkg-configuration
)

# targets
if(REGISTRY_CHECKOUT)
    set(all_ports "")
    configure_file("config/update-version.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/update-version.cmake" @ONLY)
    file(GLOB ports RELATIVE "${REGISTRY_CHECKOUT}/ports" "${REGISTRY_CHECKOUT}/ports/*")
    set(vcpkg_install_options "")
    if(VCPKG_INSTALL_DEBUG)
        list(APPEND vcpkg_install_options --debug)
    endif()
    if(VCPKG_INSTALL_EDITABLE)
        list(APPEND vcpkg_install_options --editable)
    endif()
    foreach(port IN LISTS ports)
        file(GLOB sources
          "${REGISTRY_CHECKOUT}/ports/${port}/portfile.cmake"
          "${REGISTRY_CHECKOUT}/ports/${port}/vcpkg.json"
        )
        if(sources MATCHES ";")
            string(APPEND all_ports "${port}\n")
        endif()
        add_custom_target("${port}"
          COMMAND ${vcpkg} remove --recurse "${port}" @options
          COMMAND "${CMAKE_COMMAND}" -E copy "${REGISTRY_CHECKOUT}/ports/${port}/vcpkg.json" "${CMAKE_CURRENT_BINARY_DIR}/${port}-vcpkg.json"
          COMMAND ${vcpkg} format-manifest "${CMAKE_CURRENT_BINARY_DIR}/${port}-vcpkg.json"
          COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_CURRENT_BINARY_DIR}/${port}-vcpkg.json" "${REGISTRY_CHECKOUT}/ports/${port}/vcpkg.json"
          COMMAND "${CMAKE_COMMAND}" -E remove "${CMAKE_CURRENT_BINARY_DIR}/${port}-vcpkg.json"
          COMMAND "${CMAKE_COMMAND}" "-DPORT=${port}" -P "${CMAKE_CURRENT_BINARY_DIR}/update-version.cmake"
		  COMMAND ${vcpkg} install ${vcpkg_install_options} "${port}" @options
          WORKING_DIRECTORY "${VCPKG_ROOT}"
          SOURCES ${sources}
          DEPENDS vcpkg
        )
    endforeach()
    file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/all_ports" "${all_ports}")
    add_custom_target(all-ports
      COMMAND ${vcpkg} remove --recurse @options @all_ports
      COMMAND ${vcpkg} install @options @all_ports
      WORKING_DIRECTORY "${VCPKG_ROOT}"
      SOURCES "${REGISTRY_CHECKOUT}/versions/baseline.json"
      DEPENDS vcpkg
    )
else()
    message(WARNING "You must configure `REGISTRY_...` parameters do get useful targets.")
endif()
