# CMakeLists.txt for building mcuboot as a Zephyr project
#
# Copyright (c) 2017 Open Source Foundries Limited
#
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.13.1)

# Enable Zephyr runner options which request mass erase if so
# configured.
#
# Note that this also disables the default "leave" option when
# targeting STM32 DfuSe devices with dfu-util, making the chip stay in
# the bootloader after flashing.
#
# That's the right thing, because mcuboot has nothing to do since the
# chip was just erased. The next thing the user is going to want to do
# is flash the application. (Developers can reset DfuSE devices
# manually to test mcuboot behavior on an otherwise erased flash
# device.)
macro(app_set_runner_args)
  if(CONFIG_ZEPHYR_TRY_MASS_ERASE)
    board_runner_args(dfu-util "--dfuse-modifiers=force:mass-erase")
    board_runner_args(pyocd "--flash-opt=-e=chip")
    board_runner_args(nrfjprog "--erase")
  endif()
endmacro()

# find_package(Zephyr) in order to load application boilerplate:
# http://docs.zephyrproject.org/application/application.html
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(NONE)

# Path to "boot" subdirectory of repository root.
get_filename_component(BOOT_DIR ${APPLICATION_SOURCE_DIR} DIRECTORY)
# Path to top-level repository root directory.
get_filename_component(MCUBOOT_DIR ${BOOT_DIR} DIRECTORY)
# Path to tinycrypt library source subdirectory of MCUBOOT_DIR.
set(TINYCRYPT_DIR "${MCUBOOT_DIR}/ext/tinycrypt/lib")
assert_exists(TINYCRYPT_DIR)
set(TINYCRYPT_SHA512_DIR "${MCUBOOT_DIR}/ext/tinycrypt-sha512/lib")
assert_exists(TINYCRYPT_SHA512_DIR)
# Path to crypto-fiat
set(FIAT_DIR "${MCUBOOT_DIR}/ext/fiat")
assert_exists(FIAT_DIR)
# Path to mbed-tls' asn1 parser library.
set(MBEDTLS_ASN1_DIR "${MCUBOOT_DIR}/ext/mbedtls-asn1")
assert_exists(MBEDTLS_ASN1_DIR)
set(MCUBOOT_NRF_EXT_DIR "${MCUBOOT_DIR}/ext/nrf")

if(CONFIG_BOOT_USE_NRF_CC310_BL)
  if(NOT EXISTS ${ZEPHYR_NRFXLIB_MODULE_DIR})
    message(FATAL_ERROR "
    ------------------------------------------------------------------------
    No such file or directory: ${ZEPHYR_NRFXLIB_MODULE_DIR}
    The current configuration enables nRF CC310 crypto accelerator hardware
    with the `CONFIG_BOOT_USE_NRF_CC310_BL` option. Please follow
    `ext/nrf/README.md` guide to fix your setup or use tinycrypt instead of
    the HW accelerator.
    To use the tinycrypt set `CONFIG_BOOT_ECDSA_TINYCRYPT` to y.
    ------------------------------------------------------------------------")
  endif()
endif()

zephyr_library_include_directories(
  include
  targets
  )
if(EXISTS targets/${BOARD}.h)
  zephyr_library_compile_definitions(MCUBOOT_TARGET_CONFIG="${BOARD}.h")
endif()

# Zephyr port-specific sources.
zephyr_library_sources(
  main.c
  flash_map_extended.c
  os.c
  keys.c
  )

if(DEFINED CONFIG_ENABLE_MGMT_PERUSER)
  zephyr_library_sources(
    boot_serial_extensions.c
    )
endif()

if(NOT DEFINED CONFIG_FLASH_PAGE_LAYOUT)
  zephyr_library_sources(
    flash_map_legacy.c
    )
endif()

# Generic bootutil sources and includes.
zephyr_library_include_directories(${BOOT_DIR}/bootutil/include)
zephyr_library_sources(
  ${BOOT_DIR}/bootutil/src/image_validate.c
  ${BOOT_DIR}/bootutil/src/tlv.c
  ${BOOT_DIR}/bootutil/src/encrypted.c
  ${BOOT_DIR}/bootutil/src/image_rsa.c
  ${BOOT_DIR}/bootutil/src/image_ecdsa.c
  ${BOOT_DIR}/bootutil/src/image_ed25519.c
  ${BOOT_DIR}/bootutil/src/bootutil_misc.c
  ${BOOT_DIR}/bootutil/src/fault_injection_hardening.c
  )

# library which might be common source code for MCUBoot and an application
zephyr_link_libraries(MCUBOOT_BOOTUTIL)

if(CONFIG_BOOT_FIH_PROFILE_HIGH)
zephyr_library_sources(
  ${BOOT_DIR}/bootutil/src/fault_injection_hardening_delay_rng_mbedtls.c
  )
endif()

if(CONFIG_SINGLE_APPLICATION_SLOT)
zephyr_library_sources(
  ${BOOT_DIR}/zephyr/single_loader.c
  )
zephyr_library_include_directories(${BOOT_DIR}/bootutil/src)
else()
zephyr_library_sources(
  ${BOOT_DIR}/bootutil/src/loader.c
  ${BOOT_DIR}/bootutil/src/swap_misc.c
  ${BOOT_DIR}/bootutil/src/swap_scratch.c
  ${BOOT_DIR}/bootutil/src/swap_move.c
  ${BOOT_DIR}/bootutil/src/caps.c
  )
endif()

if(CONFIG_BOOT_SIGNATURE_TYPE_ECDSA_P256 OR CONFIG_BOOT_ENCRYPT_EC256)
  zephyr_library_include_directories(
    ${MBEDTLS_ASN1_DIR}/include
    )
  zephyr_library_sources(
    # Additionally pull in just the ASN.1 parser from mbedTLS.
    ${MBEDTLS_ASN1_DIR}/src/asn1parse.c
    ${MBEDTLS_ASN1_DIR}/src/platform_util.c
    )
  if(CONFIG_BOOT_USE_TINYCRYPT)
  # When using ECDSA signatures, pull in our copy of the tinycrypt library.
  zephyr_library_include_directories(
    ${BOOT_DIR}/zephyr/include
    ${TINYCRYPT_DIR}/include
    )
  zephyr_include_directories(${TINYCRYPT_DIR}/include)

  zephyr_library_sources(
    ${TINYCRYPT_DIR}/source/ecc.c
    ${TINYCRYPT_DIR}/source/ecc_dsa.c
    ${TINYCRYPT_DIR}/source/sha256.c
    ${TINYCRYPT_DIR}/source/utils.c
    )
  elseif(CONFIG_BOOT_USE_NRF_CC310_BL)
    zephyr_library_sources(${MCUBOOT_NRF_EXT_DIR}/cc310_glue.c)
    zephyr_library_include_directories(${MCUBOOT_NRF_EXT_DIR})
    zephyr_link_libraries(nrfxlib_crypto)
  elseif(CONFIG_BOOT_USE_NRF_EXTERNAL_CRYPTO)
    zephyr_include_directories(${BL_CRYPTO_DIR}/../include)
  endif()

  # Since here we are not using Zephyr's mbedTLS but rather our own, we need
  # to set MBEDTLS_CONFIG_FILE ourselves. When using Zephyr's copy, this
  # variable is set by its Kconfig in the Zephyr codebase.
  zephyr_library_compile_definitions(
    MBEDTLS_CONFIG_FILE="${CMAKE_CURRENT_LIST_DIR}/include/mcuboot-mbedtls-cfg.h"
    )
elseif(CONFIG_BOOT_SIGNATURE_TYPE_NONE)
  zephyr_library_include_directories(
    ${BOOT_DIR}/zephyr/include
    ${TINYCRYPT_DIR}/include
    )

  zephyr_library_sources(
    ${TINYCRYPT_DIR}/source/sha256.c
    ${TINYCRYPT_DIR}/source/utils.c
    )
elseif(CONFIG_BOOT_SIGNATURE_TYPE_RSA)
  # Use mbedTLS provided by Zephyr for RSA signatures. (Its config file
  # is set using Kconfig.)
  zephyr_include_directories(include)
  if(CONFIG_BOOT_ENCRYPT_RSA)
    set_source_files_properties(
      ${BOOT_DIR}/bootutil/src/encrypted.c
      PROPERTIES
      INCLUDE_DIRECTORIES ${ZEPHYR_MBEDTLS_MODULE_DIR}/library
      )
  endif()
elseif(CONFIG_BOOT_SIGNATURE_TYPE_ED25519 OR CONFIG_BOOT_ENCRYPT_X25519)
  if(CONFIG_BOOT_USE_TINYCRYPT)
    zephyr_library_include_directories(
      ${MBEDTLS_ASN1_DIR}/include
      ${BOOT_DIR}/zephyr/include
      ${TINYCRYPT_DIR}/include
      ${TINYCRYPT_SHA512_DIR}/include
      )
    zephyr_library_sources(
      ${TINYCRYPT_DIR}/source/sha256.c
      ${TINYCRYPT_DIR}/source/utils.c
      ${TINYCRYPT_SHA512_DIR}/source/sha512.c
      # Additionally pull in just the ASN.1 parser from mbedTLS.
      ${MBEDTLS_ASN1_DIR}/src/asn1parse.c
      ${MBEDTLS_ASN1_DIR}/src/platform_util.c
      )
    zephyr_library_compile_definitions(
      MBEDTLS_CONFIG_FILE="${CMAKE_CURRENT_LIST_DIR}/include/mcuboot-mbedtls-cfg.h"
      )
  else()
    zephyr_include_directories(include)
  endif()

  zephyr_library_include_directories(
    ${BOOT_DIR}/zephyr/include
    ${FIAT_DIR}/include/
  )

  zephyr_library_sources(
    ${FIAT_DIR}/src/curve25519.c
  )
endif()

if(CONFIG_BOOT_ENCRYPT_EC256 OR CONFIG_BOOT_ENCRYPT_X25519)
  zephyr_library_sources(
    ${TINYCRYPT_DIR}/source/aes_encrypt.c
    ${TINYCRYPT_DIR}/source/aes_decrypt.c
    ${TINYCRYPT_DIR}/source/ctr_mode.c
    ${TINYCRYPT_DIR}/source/hmac.c
    ${TINYCRYPT_DIR}/source/ecc_dh.c
    )
endif()

if(CONFIG_BOOT_ENCRYPT_EC256)
  zephyr_library_sources(
    ${TINYCRYPT_DIR}/source/ecc_dh.c
    )
endif()

if(CONFIG_MCUBOOT_SERIAL)
  zephyr_sources(${BOOT_DIR}/zephyr/serial_adapter.c)
  zephyr_sources(${BOOT_DIR}/boot_serial/src/boot_serial.c)
  zephyr_sources(${BOOT_DIR}/boot_serial/src/zcbor_bulk.c)

  zephyr_include_directories(${BOOT_DIR}/bootutil/include)
  zephyr_include_directories(${BOOT_DIR}/boot_serial/include)
  zephyr_include_directories(include)

  zephyr_include_directories_ifdef(
    CONFIG_BOOT_ERASE_PROGRESSIVELY
    ${BOOT_DIR}/bootutil/src
    )

  if(CONFIG_BOOT_ENCRYPT_IMAGE)
    zephyr_library_sources(
      ${BOOT_DIR}/boot_serial/src/boot_serial_encryption.c
    )
  endif()
endif()

if(NOT CONFIG_BOOT_SIGNATURE_KEY_FILE STREQUAL "")
  # CONF_FILE points to the KConfig configuration files of the bootloader.
  foreach (filepath ${CONF_FILE})
    file(READ ${filepath} temp_text)
    string(FIND "${temp_text}" ${CONFIG_BOOT_SIGNATURE_KEY_FILE} match)
    if (${match} GREATER_EQUAL 0)
      if (NOT DEFINED CONF_DIR)
        get_filename_component(CONF_DIR ${filepath} DIRECTORY)
      else()
        message(FATAL_ERROR "Signature key file defined in multiple conf files")
      endif()
    endif()
  endforeach()

  if(IS_ABSOLUTE ${CONFIG_BOOT_SIGNATURE_KEY_FILE})
    set(KEY_FILE ${CONFIG_BOOT_SIGNATURE_KEY_FILE})
  elseif((DEFINED CONF_DIR) AND
	 (EXISTS ${CONF_DIR}/${CONFIG_BOOT_SIGNATURE_KEY_FILE}))
    set(KEY_FILE ${CONF_DIR}/${CONFIG_BOOT_SIGNATURE_KEY_FILE})
  else()
    set(KEY_FILE ${MCUBOOT_DIR}/${CONFIG_BOOT_SIGNATURE_KEY_FILE})
  endif()
  message("MCUBoot bootloader key file: ${KEY_FILE}")

  set_property(
	GLOBAL
	PROPERTY
	KEY_FILE
	${KEY_FILE}
	)

  set(GENERATED_PUBKEY ${ZEPHYR_BINARY_DIR}/autogen-pubkey.c)
  add_custom_command(
    OUTPUT ${GENERATED_PUBKEY}
    COMMAND
    ${PYTHON_EXECUTABLE}
    ${MCUBOOT_DIR}/scripts/imgtool.py
    getpub
    -k
    ${KEY_FILE}
    > ${GENERATED_PUBKEY}
    DEPENDS ${KEY_FILE}
    )
  zephyr_library_sources(${GENERATED_PUBKEY})
endif()

if(CONFIG_BOOT_ENCRYPTION_KEY_FILE AND NOT CONFIG_BOOT_ENCRYPTION_KEY_FILE STREQUAL "")
  # CONF_FILE points to the KConfig configuration files of the bootloader.
  unset(CONF_DIR)
  foreach(filepath ${CONF_FILE})
    file(READ ${filepath} temp_text)
    string(FIND "${temp_text}" ${CONFIG_BOOT_ENCRYPTION_KEY_FILE} match)
    if(${match} GREATER_EQUAL 0)
      if(NOT DEFINED CONF_DIR)
        get_filename_component(CONF_DIR ${filepath} DIRECTORY)
      else()
        message(FATAL_ERROR "Encryption key file defined in multiple conf files")
      endif()
    endif()
  endforeach()

  if(IS_ABSOLUTE ${CONFIG_BOOT_ENCRYPTION_KEY_FILE})
    set(KEY_FILE ${CONFIG_BOOT_ENCRYPTION_KEY_FILE})
  elseif((DEFINED CONF_DIR) AND
	 (EXISTS ${CONF_DIR}/${CONFIG_BOOT_ENCRYPTION_KEY_FILE}))
    set(KEY_FILE ${CONF_DIR}/${CONFIG_BOOT_ENCRYPTION_KEY_FILE})
  else()
    set(KEY_FILE ${MCUBOOT_DIR}/${CONFIG_BOOT_ENCRYPTION_KEY_FILE})
  endif()
  message("MCUBoot bootloader encryption key file: ${KEY_FILE}")

  set(GENERATED_ENCKEY ${ZEPHYR_BINARY_DIR}/autogen-enckey.c)
  add_custom_command(
    OUTPUT ${GENERATED_ENCKEY}
    COMMAND
    ${PYTHON_EXECUTABLE}
    ${MCUBOOT_DIR}/scripts/imgtool.py
    getpriv
    -k
    ${KEY_FILE}
    > ${GENERATED_ENCKEY}
    DEPENDS ${KEY_FILE}
    )
  zephyr_library_sources(${GENERATED_ENCKEY})
endif()

if(CONFIG_MCUBOOT_CLEANUP_ARM_CORE)
zephyr_library_sources(
  ${BOOT_DIR}/zephyr/arm_cleanup.c
)
endif()

if(CONFIG_MCUBOOT_NRF_CLEANUP_PERIPHERAL OR CONFIG_MCUBOOT_CLEANUP_NONSECURE_RAM)
zephyr_library_sources(
  ${BOOT_DIR}/zephyr/nrf_cleanup.c
)
endif()

if(SYSBUILD AND CONFIG_PCD_APP)
    # Sysbuild requires details of the RAM flash device are stored to the cache of MCUboot so
    # that they can be read when running partition manager
    dt_nodelabel(ram_flash_dev NODELABEL flash_sim0)
    dt_reg_addr(ram_flash_addr PATH ${ram_flash_dev})
    dt_reg_size(ram_flash_size PATH ${ram_flash_dev})

    set(RAM_FLASH_ADDR "${ram_flash_addr}" CACHE STRING "" FORCE)
    set(RAM_FLASH_SIZE "${ram_flash_size}" CACHE STRING "" FORCE)
endif()
