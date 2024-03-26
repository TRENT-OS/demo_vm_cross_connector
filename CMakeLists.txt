#
# Demo VM Cross Connector
#
# Copyright (C) 2023-2024, HENSOLDT Cyber GmbH
# 
# SPDX-License-Identifier: BSD-2-Clause
#

cmake_minimum_required(VERSION 3.8.2)

project(demo_vm_cross_connector C)

#-------------------------------------------------------------------------------
# use the SDK
find_package(os-sdk REQUIRED)
os_sdk_set_defaults()

# Kernel settings
set(KernelArmHypervisorSupport ON CACHE BOOL "" FORCE)
# set(KernelArmVtimerUpdateVOffset OFF CACHE BOOL "" FORCE)
# set(KernelArmDisableWFIWFETraps ON CACHE BOOL "" FORCE)
set(VmPCISupport ON CACHE BOOL "" FORCE)
set(LibUSB OFF CACHE BOOL "" FORCE)
set(VmInitRdFile ON CACHE BOOL "" FORCE)
# set(VmVirtioConsole ON CACHE BOOL "" FORCE)

include("${CMAKE_CURRENT_LIST_DIR}/plat/${PLATFORM}/plat.cmake" OPTIONAL)
os_sdk_setup()
CAmkESAddCPPInclude("plat/${PLATFORM}")
os_sdk_import_camkes_vm()
os_sdk_import_from_global_components(
    "components/FileServer"
    "components/VirtQueue"
)

#-------------------------------------------------------------------------------
# config
AddCamkesCPPFlag(cpp_flags CONFIG_VARS VmEmmc2NoDMA VmVchan VmVirtioNetVirtqueue)

if(NOT KernelPlatformQEMUArmVirt)
    AddCamkesCPPFlag(cpp_flags CONFIG_VARS VmVUSB)
endif()

if(KernelPlatformTK1)
    AddCamkesCPPFlag(cpp_flags CONFIG_VARS Tk1DeviceFwd Tk1Insecure)
endif()


#-------------------------------------------------------------------------------
# VM Images

list(
    APPEND CMAKE_MODULE_PATH
    "${CMAKE_CURRENT_LIST_DIR}/camkes-vm-images"
    "${CMAKE_CURRENT_LIST_DIR}/camkes-vm-linux"
)
find_package(camkes-vm-images REQUIRED)
find_package(camkes-vm-linux REQUIRED)

include("${CAMKES_VM_LINUX_HELPERS_PATH}")

# images for the selected platform
set(VM_IMG_LINUX "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/linux")

if(BUILD_CROSSVM)
    set(VM_IMG_ROOTFS "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/rootfs.cpio.gz")
    # Checkout and configure linux to build crossvm module
    ExternalProject_Add(
        checkout_linux
        GIT_REPOSITORY
        ${linux_repo}
        GIT_TAG
        ${linux_tag}
        GIT_SHALLOW
        1
        GIT_PROGRESS
        1
        BUILD_COMMAND
        ""
        INSTALL_COMMAND
        ""
        CONFIGURE_COMMAND
        ""
        SOURCE_DIR
        ${CMAKE_CURRENT_BINARY_DIR}/linux_out
    )
    # Linux config and symvers are to be copied to unpacked archive
    set(linux_config "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/linux_configs/config")
    set(linux_symvers "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/linux_configs/Module.symvers")
    # Configure unpacked archive with config and symvers
    ConfigureLinux(
        ${CMAKE_CURRENT_BINARY_DIR}/linux_out
        ${linux_config}
        ${linux_symvers}
        configure_vm_linux
        ARCH
        ${linux_arch}
        CROSS_COMPILE
        ${linux_cross_compile}
        DEPENDS
        checkout_linux
    )

    # Compile CrossVM Dataport Module
    DefineLinuxModule(
        ${CAMKES_VM_LINUX_DIR}/camkes-linux-artifacts/camkes-linux-modules/camkes-connector-modules/connection
        output_module
        output_module_target
        KERNEL_DIR
        ${CMAKE_CURRENT_BINARY_DIR}/linux_out
        ARCH
        ${linux_arch}
        CROSS_COMPILE
        ${linux_cross_compile}
        DEPENDS
        checkout_linux
        configure_vm_linux
    )
    AddFileToOverlayDir(
        "connection.ko"
        ${output_module}
        "lib/modules/4.14.87/kernel/drivers/vmm"
        overlay
        DEPENDS
        output_module_target
    )

    # Complile CrossVM Dataport Apps
    ExternalProject_Add(
        dataport-apps
        URL
        file:///${CAMKES_VM_LINUX_DIR}/camkes-linux-artifacts/camkes-linux-apps/camkes-connector-apps/pkgs/dataport
        BINARY_DIR
        ${CMAKE_CURRENT_LIST_DIR}/dataport_apps
        INSTALL_COMMAND
        ""
        BUILD_ALWAYS
        ON
        EXCLUDE_FROM_ALL
        CMAKE_ARGS
        -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
    )
    AddExternalProjFilesToOverlay(
        dataport-apps
        ${CMAKE_CURRENT_LIST_DIR}/dataport_apps
        overlay
        "usr/bin"
        FILES
        dataport_read
        dataport_write
    )

    # Complile CrossVM Event Apps
    ExternalProject_Add(
        event-apps
        URL
        file:///${CAMKES_VM_LINUX_DIR}/camkes-linux-artifacts/camkes-linux-apps/camkes-connector-apps/pkgs/emits_event
        BINARY_DIR
        ${CMAKE_CURRENT_LIST_DIR}/emits_event_apps
        INSTALL_COMMAND
        ""
        BUILD_ALWAYS
        ON
        EXCLUDE_FROM_ALL
        CMAKE_ARGS
        -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
    )
    AddExternalProjFilesToOverlay(
        event-apps
        ${CMAKE_CURRENT_LIST_DIR}/emits_event_apps
        overlay
        "usr/bin"
        FILES
        emits_event_emit
    )

    # Complile CrossVM Consume Event Apps
    ExternalProject_Add(
        consume-event-apps
        URL
        file:///${CAMKES_VM_LINUX_DIR}/camkes-linux-artifacts/camkes-linux-apps/camkes-connector-apps/pkgs/consumes_event
        BINARY_DIR
        ${CMAKE_CURRENT_LIST_DIR}/consume_event_apps
        INSTALL_COMMAND
        ""
        BUILD_ALWAYS
        ON
        EXCLUDE_FROM_ALL
        CMAKE_ARGS
        -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
    )
    AddExternalProjFilesToOverlay(
        consume-event-apps
        ${CMAKE_CURRENT_LIST_DIR}/consume_event_apps
        overlay
        "usr/bin"
        FILES
        consumes_event_wait
    )

    # Add script to initialise dataport module
    AddFileToOverlayDir(
        "S90crossvm_module_init"
        ${CMAKE_CURRENT_LIST_DIR}/overlay_files/init_scripts/cross_vm_module_init
        "etc/init.d"
        overlay
    )
else()
    # User pre-configured rootfs file with crossvm modules and apps installed
    set(VM_IMG_ROOTFS "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/rootfs_crossvm.cpio.gz")
endif()

# Add script to test dataport module
AddFileToOverlayDir(
    "S91crossvm_test"
    ${CMAKE_CURRENT_LIST_DIR}/overlay_files/init_scripts/cross_vm_test
    "etc/init.d"
    overlay
)

# Construct new rootfs
AddOverlayDirToRootfs(
    overlay
    ${VM_IMG_ROOTFS}
    "buildroot"
    "rootfs_install"
    output_overlayed_rootfs_location
    rootfs_target
    GZIP
)

# DefineCAmkESVMFileServer() defines a CAmkESVMFileServer using the declared
# no explicit dependency to "rootfs_target" required, because implicit dependency on "output_overlayed_rootfs_location" is sufficient
DefineCAmkESVMFileServer(
    INSTANCE "fileServer"
    FILES
        "linux:${VM_IMG_LINUX}"
        "linux-initrd:${output_overlayed_rootfs_location}"
    # DEPENDS
    #     rootfs_target
)

# # fileserver images and dependencies. Images are placed into a CPIO archive.
# get_target_property(FS_FILES vm_fserver_config FILES)
# get_target_property(FS_DEPS vm_fserver_config DEPS)
# # Build CPIO archive given the defined kernel and rootfs images
# include(cpio)
# MakeCPIO("file_server_archive.o" "${FS_FILES}" DEPENDS "${FS_DEPS}")
# add_library(fileserver_cpio STATIC EXCLUDE_FROM_ALL file_server_archive.o)
# set_property(TARGET fileserver_cpio PROPERTY LINKER_LANGUAGE C)
# ExtendCAmkESComponentInstance(FileServer fileServer LIBS fileserver_cpio)


# CAmkESAddCPPInclude("${SDK_SEL4_CAMKES_DIR}/libs/sel4_projects_libs/libsel4vm/include/sel4vm")

#-------------------------------------------------------------------------------

#CAmkESAddImportPath(${KernelARMPlatform})

# Declare root server
#DeclareCAmkESRootserver(
#    vm_minimal.camkes
#    CPP_FLAGS
#        ${cpp_flags}
#    CPP_INCLUDES
#        ${CAMKES_VM_DIR}/components/VM_Arm
#)

# Define our VM Component with out cross vm dataports glue code
DeclareCAmkESComponent(
    VM SOURCES
    SOURCES
        src/cross_vm_connections.c
    C_FLAGS
        -Wall
)

# Define our demo component that shares a dataport with the VM
DeclareCAmkESComponent(
    CrossvmInit
    SOURCES
        components/CrossvmInit/crossvm_init.c
    C_FLAGS
        -Wall
)

os_sdk_create_CAmkES_system("vm_cross_connector.camkes")
