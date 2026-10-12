cmake_minimum_required(VERSION 3.15)

project(dls_messages)
cmake_policy(SET CMP0048 NEW)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

add_compile_options("-std=c++2a")

# Two libraries are created:
#   - dls_messages_default_ros2: messages generated from idls/ros2_interface
#   - dls_messages: DLS custom messages. It links publicly dls_messages_default_ros2, so consumers linking dls_messages get both.
# DLS_MESSAGES_ONLY_ROS2: build and install only dls_messages_default_ros2 (used to install the ROS2 messages in the docker image).
# DLS_MESSAGES_USE_INSTALLED_ROS2: build and install only dls_messages, linking the already installed dls_messages_default_ros2.
option(DLS_MESSAGES_ONLY_ROS2 "Build only the dls_messages_default_ros2 library" OFF)
option(DLS_MESSAGES_USE_INSTALLED_ROS2 "Build only dls_messages and link the installed dls_messages_default_ros2" ON)
if(DLS_MESSAGES_ONLY_ROS2 AND DLS_MESSAGES_USE_INSTALLED_ROS2)
    message(FATAL_ERROR "DLS_MESSAGES_ONLY_ROS2 and DLS_MESSAGES_USE_INSTALLED_ROS2 cannot be both ON")
endif()

set(BUILD_DLS_LIBRARY ON)
set(BUILD_ROS2_LIBRARY ON)
if(DLS_MESSAGES_ONLY_ROS2)
    set(BUILD_DLS_LIBRARY OFF)
elseif(DLS_MESSAGES_USE_INSTALLED_ROS2)
    set(BUILD_ROS2_LIBRARY OFF)
endif()


# ================================== Messages =================================
include(generate_msg_library)
include(CMakePackageConfigHelpers)
set (LIBRARY_NAME dls_messages)
set (ROS2_LIBRARY_NAME dls_messages_default_ros2)

# iterate over all the idl files that are in the idls directory and split them between ros2 and dls idls.
file(GLOB_RECURSE idl_files CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/idls/*.idl"
)
set(ros2_idl_files ${idl_files})
list(FILTER ros2_idl_files INCLUDE REGEX "^${CMAKE_CURRENT_SOURCE_DIR}/idls/ros2_interface/")
set(dls_idl_files ${idl_files})
list(FILTER dls_idl_files EXCLUDE REGEX "^${CMAKE_CURRENT_SOURCE_DIR}/idls/ros2_interface/")

set(idl_files "")
if(BUILD_ROS2_LIBRARY)
    list(APPEND idl_files ${ros2_idl_files})
endif()
if(BUILD_DLS_LIBRARY)
    list(APPEND idl_files ${dls_idl_files})
endif()

# generate msg files using fastddsgen
foreach(idl_file IN LISTS idl_files)
    fastddsgen_trigger(${idl_file})
endforeach()

# Create a custom target that depends on all the fastddsgen targets, so that we can wait for all the generated files to be created before building the cpp and python libraries.
add_custom_target(dls_messages_fastddsgen_all
    DEPENDS ${DLS_MESSAGES_FASTDDSGEN_TARGETS}
)

# install the library, its exported targets and its package config file
function(install_msg_library library_name)
    configure_package_config_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${library_name}Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${library_name}Config.cmake
        INSTALL_DESTINATION cmake
        NO_SET_AND_CHECK_MACRO
        NO_CHECK_REQUIRED_COMPONENTS_MACRO
    )
    install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/${library_name}Config.cmake
        DESTINATION cmake
        COMPONENT dls_messages_dev
    )
    install(TARGETS ${library_name}
        EXPORT ${library_name}Targets
        LIBRARY
            DESTINATION /usr/lib
            COMPONENT dls_messages_dev
        ARCHIVE
            DESTINATION /usr/lib
            COMPONENT dls_messages_dev
        RUNTIME
            DESTINATION ${DLS_INSTALL_RUNTIME_DIR}
            COMPONENT dls_messages_dev
    )
    install(
        EXPORT ${library_name}Targets
        FILE ${library_name}Targets.cmake
        NAMESPACE dls_messages::
        DESTINATION cmake
        COMPONENT dls_messages_dev
    )
endfunction()

# ROS2 messages library
if(BUILD_ROS2_LIBRARY)
    add_library(${ROS2_LIBRARY_NAME} SHARED)
    add_library(dls_messages::${ROS2_LIBRARY_NAME} ALIAS ${ROS2_LIBRARY_NAME})
    add_dependencies(${ROS2_LIBRARY_NAME}
        dls_messages_fastddsgen_all
    )
    # add cpp generated files to the library and create python bindings for the generated cpp files
    foreach(idl_file IN LISTS ros2_idl_files)
        generate_msg_library(${idl_file} ${ROS2_LIBRARY_NAME})
    endforeach()

    target_include_directories(${ROS2_LIBRARY_NAME}
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include/dls_messages/dds/ros2_interface>
            $<INSTALL_INTERFACE:/usr/include>
            $<INSTALL_INTERFACE:/usr/include/dls_messages/dds/ros2_interface>
    )
    target_link_libraries(${ROS2_LIBRARY_NAME}
        PUBLIC
            fastcdr
            fastdds
    )
    install_msg_library(${ROS2_LIBRARY_NAME})
else()
    message(STATUS "dls_messages: using installed ${ROS2_LIBRARY_NAME}")
    find_package(${ROS2_LIBRARY_NAME} REQUIRED
        HINTS /usr/cmake /usr/local/cmake
    )
endif()

# DLS custom messages library
if(BUILD_DLS_LIBRARY)
    add_library(${LIBRARY_NAME} SHARED)
    add_dependencies(${LIBRARY_NAME}
        dls_messages_fastddsgen_all
    )
    # add cpp generated files to the library and create python bindings for the generated cpp files
    foreach(idl_file IN LISTS dls_idl_files)
        generate_msg_library(${idl_file} ${LIBRARY_NAME})
    endforeach()

    target_include_directories(${LIBRARY_NAME}
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
            $<INSTALL_INTERFACE:/usr/include>
    )
    target_link_libraries(${LIBRARY_NAME}
        PUBLIC
            fastcdr
            fastdds
            dls_messages::${ROS2_LIBRARY_NAME}
    )
    # dls custom messages do not use ros2 symbols, so with --as-needed (Ubuntu default) the linker would drop
    # dls_messages_default_ros2. Force it in DT_NEEDED so that linking dls_messages always carries it.
    target_link_options(${LIBRARY_NAME}
        PRIVATE
            "LINKER:--no-as-needed"
    )
    install_msg_library(${LIBRARY_NAME})
else()
    message(STATUS "dls_messages: building only ${ROS2_LIBRARY_NAME}")
endif()

install(
    DIRECTORY
		${CMAKE_CURRENT_LIST_DIR}/cmake/
    DESTINATION
		cmake
	COMPONENT
		dls_messages_dev
)
install(
    DIRECTORY
        ${CMAKE_CURRENT_BINARY_DIR}/include/${LIBRARY_NAME}
    DESTINATION
        /usr/include/
    COMPONENT
        dls_messages_dev
)

# set(CPACK_PACKAGE_VERSION_MAJOR         ${PROJECT_VERSION_MAJOR})
# set(CPACK_PACKAGE_VERSION_MINOR         ${PROJECT_VERSION_MINOR})
# set(CPACK_PACKAGE_VERSION_PATCH         ${PROJECT_VERSION_PATCH})
# set(CPACK_DEBIAN_PACKAGE_VERSION        ${PROJECT_VERSION})
set(CPACK_PACKAGE_VENDOR                "dlslab")
set(CPACK_PACKAGE_CONTACT               "dlslab")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER     "dlslab")
set(CPACK_VERBATIM_VARIABLES            ON)
set(CPACK_GENERATOR                     "DEB")
set(CPACK_DEB_COMPONENT_INSTALL         ON)
set(CPACK_PACKAGE_NAME 					"dls_messages_dev")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY 	"dls_messages_dev")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS      ON)

set(CPACK_DEBIAN_DLS_MESSAGES_PACKAGE_NAME          "dls-messages" CACHE INTERNAL "")
set(CPACK_DEBIAN_DLS_MESSAGES_FILE_NAME             "dls-messages.deb" CACHE INTERNAL "")  
set(CPACK_DEBIAN_DLS_MESSAGES_PACKAGE_DEPENDS       "" CACHE INTERNAL "")
set(CPACK_DEBIAN_DLS_MESSAGES_PACKAGE_SHLIBDEPS     ON)

set(CPACK_DEBIAN_DLS_MESSAGES_DEV_PACKAGE_NAME      "dls-messages-dev" CACHE INTERNAL "")
set(CPACK_DEBIAN_DLS_MESSAGES_DEV_FILE_NAME         "dls-messages-dev.deb" CACHE INTERNAL "")  
set(CPACK_DEBIAN_DLS_MESSAGES_DEV_PACKAGE_DEPENDS   "" CACHE INTERNAL "")
set(CPACK_DEBIAN_DLS_MESSAGES_DEV_PACKAGE_SHLIBDEPS ON)

get_cmake_property(CPACK_COMPONENTS_ALL COMPONENTS)
list(REMOVE_ITEM CPACK_COMPONENTS_ALL "Unspecified")

include(CPack)
