# Copyright (c) 2017 The Khronos Group Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# Author:
#

#set (CMAKE_VERBOSE_MAKEFILE 1)

include(GNUInstallDirs)

# Use this feature for Windows to automatically generate an exports file for the DLL.
# See https://blog.kitware.com/create-dlls-on-windows-without-declspec-using-new-cmake-export-all-feature/
include(GenerateExportHeader)
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS false)


if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(FALLBACK_CONFIG_DIRS "/etc/xdg" CACHE STRING
        "Search path to use when XDG_CONFIG_DIRS is unset or empty or the current process is SUID/SGID. Default is freedesktop compliant.")
    set(FALLBACK_DATA_DIRS "/usr/local/share:/usr/share" CACHE STRING
        "Search path to use when XDG_DATA_DIRS is unset or empty or the current process is SUID/SGID. Default is freedesktop compliant.")
endif()
if(WIN32)
    set(openxr_loader_RESOURCE_FILE ${CMAKE_CURRENT_SOURCE_DIR}/loader.rc)
endif()

# List of all files externally generated outside of the loader that the loader
# needs to build with.
SET(LOADER_EXTERNAL_GEN_FILES
    ${COMMON_GENERATED_OUTPUT}
)
run_xr_xml_generate(loader_source_generator.py xr_generated_loader.hpp)
run_xr_xml_generate(loader_source_generator.py xr_generated_loader.cpp)

if(DYNAMIC_LOADER)
    add_definitions(-DXRAPI_DLL_EXPORT)
    set(LIBRARY_TYPE SHARED)
else() # build static lib
    set(LIBRARY_TYPE STATIC)
endif()

add_library(openxr_loader ${LIBRARY_TYPE}
    api_layer_interface.cpp
    api_layer_interface.hpp
    loader_core.cpp
    loader_instance.cpp
    loader_instance.hpp
    loader_logger.cpp
    loader_logger.hpp
    loader_logger_recorders.cpp
    loader_logger_recorders.hpp
    manifest_file.cpp
    manifest_file.hpp
    runtime_interface.cpp
    runtime_interface.hpp
    ${GENERATED_OUTPUT}
    ${OPENXR_COMMON_DIR}/filesystem_utils.cpp
    ${OPENXR_COMMON_DIR}/filesystem_utils.hpp
    ${OPENXR_COMMON_DIR}/hex_and_handles.cpp
    ${OPENXR_COMMON_DIR}/hex_and_handles.h
    ${OPENXR_COMMON_DIR}/object_info.cpp
    ${OPENXR_COMMON_DIR}/object_info.h
    ${OPENXR_SOURCE_DIR}/src/external/jsoncpp/src/lib_json/json_reader.cpp
    ${OPENXR_SOURCE_DIR}/src/external/jsoncpp/src/lib_json/json_value.cpp
    ${OPENXR_SOURCE_DIR}/src/external/jsoncpp/src/lib_json/json_writer.cpp
    ${LOADER_EXTERNAL_GEN_FILES}
    ${openxr_loader_RESOURCE_FILE}
)
set_target_properties(openxr_loader PROPERTIES FOLDER ${LOADER_FOLDER})

set_source_files_properties(
    ${LOADER_EXTERNAL_GEN_FILES}
    PROPERTIES GENERATED TRUE
)
add_dependencies(openxr_loader
    generate_openxr_header
    xr_global_generated_files
)
target_include_directories(openxr_loader
    PRIVATE
    ${OPENXR_COMMON_DIR}

    # for OpenXR headers
    ${OPENXR_SOURCE_DIR}/include
    ${OPENXR_BINARY_DIR}/include

    # for generated dispatch table
    ${CMAKE_CURRENT_SOURCE_DIR}/..
    ${CMAKE_CURRENT_BINARY_DIR}/..

    # for target-specific generated files
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}

    # For jsoncpp
    ${OPENXR_SOURCE_DIR}/src/external/jsoncpp/include
)
if(VulkanHeaders_FOUND)
    target_include_directories(openxr_loader
        PRIVATE ${Vulkan_INCLUDE_DIRS}
    )
endif()
if(NOT BUILD_LOADER_WITH_EXCEPTION_HANDLING)
    target_compile_definitions(openxr_loader
        PRIVATE XRLOADER_DISABLE_EXCEPTION_HANDLING
    )
endif()

target_compile_definitions(openxr_loader
    PRIVATE API_NAME="OpenXR"
)
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_compile_definitions(openxr_loader
        PRIVATE FALLBACK_CONFIG_DIRS="${FALLBACK_CONFIG_DIRS}"
        PRIVATE FALLBACK_DATA_DIRS="${FALLBACK_DATA_DIRS}"
        PRIVATE SYSCONFDIR="${CMAKE_INSTALL_FULL_SYSCONFDIR}"
    )
    if(NOT(CMAKE_INSTALL_FULL_SYSCONFDIR STREQUAL "/etc"))
        target_compile_definitions(openxr_loader PRIVATE EXTRASYSCONFDIR="/etc")
    endif()

    set_target_properties(openxr_loader PROPERTIES SOVERSION "${MAJOR}" VERSION "${MAJOR}.${MINOR}.${PATCH}")
    target_link_libraries(openxr_loader -lstdc++fs -ldl -lpthread -lm)

    add_custom_target(libopenxr_loader.so.${MAJOR}.${MINOR} ALL
        COMMAND ${CMAKE_COMMAND} -E create_symlink libopenxr_loader.so.${MAJOR}.${MINOR}.${PATCH} libopenxr_loader.so.${MAJOR}.${MINOR})

    set(XR_API_VERSION "${MAJOR}.${MINOR}")
    configure_file("openxr.pc.in" "openxr.pc" @ONLY)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/openxr.pc" DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
elseif(WIN32)
    # WindowsStore (UWP) apps must be compiled with dynamic CRT linkage (default)
    if (NOT CMAKE_SYSTEM_NAME STREQUAL "WindowsStore")
        foreach(configuration in CMAKE_C_FLAGS_DEBUG
                                CMAKE_C_FLAGS_RELEASE
                                CMAKE_C_FLAGS_RELWITHDEBINFO
                                CMAKE_CXX_FLAGS_DEBUG
                                CMAKE_CXX_FLAGS_RELEASE
                                CMAKE_CXX_FLAGS_RELWITHDEBINFO)
            # If building DLLs, force static CRT linkage
            if(DYNAMIC_LOADER)
                if (${configuration} MATCHES "/MD")
                    string(REGEX REPLACE "/MD" "/MT" ${configuration} "${${configuration}}")
                endif()
            else()  # Otherwise for static libs, link the CRT dynamically
                if (${configuration} MATCHES "/MT")
                    string(REGEX REPLACE "/MT" "/MD" ${configuration} "${${configuration}}")
                endif()
            endif()
        endforeach()
    endif()

    target_link_libraries(openxr_loader shlwapi)
    target_compile_options(openxr_loader PRIVATE)
    generate_export_header(openxr_loader)
    # set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS false)

    # Need to copy DLL to client directories so clients can easily load it.
    if( (DYNAMIC_LOADER) AND (CMAKE_GENERATOR MATCHES "^Visual Studio.*") )
        file( TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>/openxr_loader.dll COPY_SRC_PATH )
        file( TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/../tests/hello_xr/$<CONFIGURATION>/ COPY_DST_TEST_PATH )
        file( TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/../tests/loader_test/$<CONFIGURATION>/ COPY_DST_LOADER_TEST_PATH )
        add_custom_command( TARGET openxr_loader POST_BUILD
            COMMAND xcopy /Y /I ${COPY_SRC_PATH} ${COPY_DST_TEST_PATH}
            COMMAND xcopy /Y /I ${COPY_SRC_PATH} ${COPY_DST_LOADER_TEST_PATH}
        )
    endif()
endif()

if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID MATCHES "Clang")
    target_compile_options(openxr_loader
        PRIVATE -Wall -Wextra -Wno-unused-parameter -Wno-missing-field-initializers -Wpointer-arith
        PRIVATE -fno-strict-aliasing -fno-builtin-memcmp "$<$<COMPILE_LANGUAGE:CXX>:-fno-rtti>"
    )
    # For GCC version 7.1 or greater, we need to disable the implicit fallthrough warning since
    # there's no consistent way to satisfy all compilers until they all accept the C++17 standard
    if (CMAKE_COMPILER_IS_GNUCC AND NOT (CMAKE_CXX_COMPILER_VERSION LESS 7.1))
        target_compile_options(openxr_loader PRIVATE -Wimplicit-fallthrough=0)
    endif()
endif()

install(TARGETS openxr_loader
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
