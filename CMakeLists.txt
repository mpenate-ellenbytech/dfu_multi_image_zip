function(dfu_multi_image_zip)
  set(oneValueArgs OUTPUT TYPE TARGET IMAGE)
  set(multiValueArgs BIN_FILES SCRIPT_PARAMS DEPENDS)
  cmake_parse_arguments(GENZIP "" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

  if (NOT(
    GENZIP_BIN_FILES AND
    GENZIP_SCRIPT_PARAMS AND
    GENZIP_OUTPUT AND
    GENZIP_TYPE
    ))
    message(FATAL_ERROR "Missing required param")
  endif()

  if(SYSBUILD AND GENZIP_IMAGE)
    set(APPNAME ${GENZIP_IMAGE})
    sysbuild_get(CONFIG_BUILD_OUTPUT_META IMAGE ${GENZIP_IMAGE} VAR CONFIG_BUILD_OUTPUT_META KCONFIG)
    sysbuild_get(CONFIG_BOARD IMAGE ${GENZIP_IMAGE} VAR CONFIG_BOARD KCONFIG)
    sysbuild_get(CONFIG_SOC IMAGE ${GENZIP_IMAGE} VAR CONFIG_SOC KCONFIG)

    if(CONFIG_BUILD_OUTPUT_META)
      sysbuild_get(application_image_dir IMAGE ${GENZIP_IMAGE} VAR APPLICATION_BINARY_DIR CACHE)
      sysbuild_get(CONFIG_KERNEL_BIN_NAME IMAGE ${GENZIP_IMAGE} VAR CONFIG_KERNEL_BIN_NAME KCONFIG)
      set(meta_info_file ${GENZIP_IMAGE}_extra_byproducts)
      set(meta_argument --meta-info-file ${application_image_dir}/zephyr/${CONFIG_KERNEL_BIN_NAME}.meta)
    endif()
  else()
    get_filename_component(APPNAME ${APPLICATION_SOURCE_DIR} NAME)
    if(CONFIG_BUILD_OUTPUT_META)
      set(meta_info_file ${PROJECT_BINARY_DIR}/${KERNEL_META_NAME})
      set(meta_argument --meta-info-file ${meta_info_file})
    endif()
  endif()

  add_custom_target(
    ${GENZIP_TARGET} ALL
    COMMAND
    ${PYTHON_EXECUTABLE}
    ${ZEPHYR_NRF_MODULE_DIR}/scripts/bootloader/generate_zip.py
    --bin-files ${GENZIP_BIN_FILES}
    --output ${GENZIP_OUTPUT}
    --name "${APPNAME}"
    ${meta_argument}
    ${GENZIP_SCRIPT_PARAMS}
    "type=${GENZIP_TYPE}"
    "board=${CONFIG_BOARD}"
    "soc=${CONFIG_SOC}"
    DEPENDS ${GENZIP_DEPENDS}
    )
endfunction()

function(${SYSBUILD_CURRENT_MODULE_NAME}_post_domains)
  cmake_parse_arguments(POST_DOMAINS "" "" "IMAGES" ${ARGN})

  message("Invoking ${CMAKE_CURRENT_FUNCTION}. Images: ${POST_DOMAINS_IMAGES}.")

  if(SB_CONFIG_DFU_MULTI_IMAGE_PACKAGE_BUILD)
    include(${ZEPHYR_BASE}/../nrf/cmake/fw_zip.cmake)
    include(${ZEPHYR_BASE}/../nrf/cmake/dfu_multi_image.cmake)

    set(generate_bin_files)
    set(generate_script_params)
    set(generate_depends)

    if(SB_CONFIG_DFU_MULTI_IMAGE_PACKAGE_APP)
      sysbuild_get(${DEFAULT_IMAGE}_image_dir IMAGE ${DEFAULT_IMAGE} VAR APPLICATION_BINARY_DIR CACHE)
      sysbuild_get(${DEFAULT_IMAGE}_kernel_name IMAGE ${DEFAULT_IMAGE} VAR CONFIG_KERNEL_BIN_NAME KCONFIG)
      sysbuild_get(${DEFAULT_IMAGE}_version IMAGE ${DEFAULT_IMAGE} VAR CONFIG_MCUBOOT_IMGTOOL_SIGN_VERSION KCONFIG)

      set(app_core_binary_name "${${DEFAULT_IMAGE}_kernel_name}.signed.bin")
      list(APPEND generate_bin_files "${${DEFAULT_IMAGE}_image_dir}/zephyr/${app_core_binary_name}")
      list(APPEND generate_depends "${DEFAULT_IMAGE}_extra_byproducts")
      set(generate_script_params
        ${generate_script_params}
        "${app_core_binary_name}load_address=$<TARGET_PROPERTY:partition_manager,PM_APP_ADDRESS>"
        "${app_core_binary_name}image_index=0"
        "${app_core_binary_name}slot_index_primary=1"
        "${app_core_binary_name}slot_index_secondary=2"
        "${app_core_binary_name}version_MCUBOOT=${${DEFAULT_IMAGE}_version}"
      )
    endif()

    if(SB_CONFIG_DFU_MULTI_IMAGE_PACKAGE_NET)
      get_property(image_name GLOBAL PROPERTY DOMAIN_APP_CPUNET)
      sysbuild_get(net_core_soc IMAGE ${image_name} VAR SOC CACHE)
      sysbuild_get(net_core_version IMAGE ${image_name} VAR VERSION CACHE)
      sysbuild_get(net_core_board IMAGE ${image_name} VAR BOARD CACHE)

      set(net_core_binary_name "signed_by_mcuboot_and_b0_${image_name}.bin")
      list(APPEND generate_bin_files "${PROJECT_BINARY_DIR}/${net_core_binary_name}")
      list(APPEND generate_depends "${image_name}_signed_packaged_target")
      set(generate_script_params
        ${generate_script_params}
        "${net_core_binary_name}image_index=1"
        "${net_core_binary_name}slot_index_primary=3"
        "${net_core_binary_name}slot_index_secondary=4"
        "${net_core_binary_name}load_address=$<TARGET_PROPERTY:partition_manager,CPUNET_PM_APP_ADDRESS>"
        "${net_core_binary_name}board=${net_core_board}"
        "${net_core_binary_name}version=${net_core_version}"
        "${net_core_binary_name}soc=${net_core_soc}"
      )
    endif()


    dfu_multi_image_zip(
      IMAGE ${DEFAULT_IMAGE}
      TARGET dfu_application_zip
      OUTPUT ${PROJECT_BINARY_DIR}/dfu_application.zip
      BIN_FILES ${generate_bin_files}
      TYPE application
      SCRIPT_PARAMS ${generate_script_params}
      DEPENDS ${generate_depends}
      )
  endif()
endfunction()
