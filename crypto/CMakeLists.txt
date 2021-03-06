#
# Copyright (c) 2018 Nordic Semiconductor
#
# SPDX-License-Identifier: LicenseRef-BSD-5-Clause-Nordic
#

include(${NRFXLIB_DIR}/common.cmake)

nrfxlib_calculate_lib_path(lib_path)

zephyr_interface_library_named(nrfxlib_crypto)

if (CONFIG_NRF_OBERON)
  set(OBERON_BASE ${CMAKE_CURRENT_SOURCE_DIR}/nrf_oberon)
  set(OBERON_VER 3.0.3)
  set(OBERON_LIB ${OBERON_BASE}/${lib_path}/liboberon_${OBERON_VER}.a)
  if(NOT EXISTS ${OBERON_LIB})
    message(WARNING "This combination of SoC and floating point ABI is not supported by the nrf_oberon lib."
                    "(${OBERON_LIB} doesn't exist.)")
  endif()
  target_include_directories(${IMAGE}nrfxlib_crypto INTERFACE ${OBERON_BASE}/include)
  target_link_libraries(${IMAGE}nrfxlib_crypto INTERFACE ${OBERON_LIB})
endif()

if (CONFIG_NRF_CC310_BL)
  assert(CONFIG_HAS_HW_NRF_CC310 "nrf_cc310_bl requires CryptoCell hardware.")
  set(CC310_BL_BASE ${CMAKE_CURRENT_SOURCE_DIR}/nrf_cc310_bl)
  if (NOT CONFIG_NRF_CC310_BL_INTERRUPTS)
    set(CC310_FLAVOR no-interrupts)
  endif()
  set(CC310_BL_VER 0.9.12)
  set(CC310_BL_LIB ${CC310_BL_BASE}/${lib_path}/${CC310_FLAVOR}/libnrf_cc310_bl_${CC310_BL_VER}.a)
  if(NOT EXISTS ${CC310_BL_LIB})
    message(WARNING "This combination of SoC, floating point ABI, and interrupts settings is not supported by the nrf_cc310_bl lib."
                    "(${CC310_BL_LIB} doesn't exist.)")
  endif()
  target_include_directories(${IMAGE}nrfxlib_crypto INTERFACE ${CC310_BL_BASE}/include)
  target_link_libraries(${IMAGE}nrfxlib_crypto INTERFACE ${CC310_BL_LIB})
endif()

if (CONFIG_NRF_CC310_PLATFORM)
  set (NRF_CC310_PLATFORM_BASE ${CMAKE_CURRENT_SOURCE_DIR}/nrf_cc310_platform)
  set (NRF_CC310_PLATFORM_VER 0.9.1)
  if (NOT CONFIG_CC310_INTERRUPTS_BACKEND)
    set(CC310_PLATFORM_FLAVOR no-interrupts)
  endif()
  set(NRF_CC310_PLATFORM_LIB
    ${NRF_CC310_PLATFORM_BASE}/${lib_path}/${CC310_PLATFORM_FLAVOR}/libnrf_cc310_platform_${NRF_CC310_PLATFORM_VER}.a
  )

  if (NOT EXISTS ${NRF_CC310_PLATFORM_LIB})
    message(WARNING "This combination of SoC, floating point ABI, and interrupts settings is not supported by the nrf_cc310_platform lib."
                    "(${NRF_CC310_PLATFORM_LIB} doesn't exist.)")
  endif()
  #
  # Uses a normal add_library to avoid --whole-archive
  #
  add_library(${IMAGE}platform_cc310 STATIC IMPORTED GLOBAL)
  set_target_properties(${IMAGE}platform_cc310 PROPERTIES IMPORTED_LOCATION
      "${NRF_CC310_PLATFORM_LIB}")
  target_include_directories(${IMAGE}platform_cc310 INTERFACE
      "${NRF_CC310_PLATFORM_BASE}/include")
  zephyr_link_libraries(${IMAGE}platform_cc310)
  #
  # Add companion sources to directly to zephyr
  #
  zephyr_sources(${NRF_CC310_PLATFORM_BASE}/src/nrf_cc310_platform_abort_zephyr.c)
  zephyr_sources(${NRF_CC310_PLATFORM_BASE}/src/nrf_cc310_platform_mutex_zephyr.c)
endif()

if (CONFIG_CC310_BACKEND)
  set(NRF_CC310_BASE ${CMAKE_CURRENT_SOURCE_DIR}/nrf_cc310_mbedcrypto)
  set(NRF_CC310_VER 0.9.1)
  if (NOT CONFIG_CC310_INTERRUPTS_BACKEND)
    set(CC310_FLAVOR no-interrupts)
  endif()
  set(NRF_CC310_LIB
    ${NRF_CC310_BASE}/${lib_path}/${CC310_FLAVOR}/libnrf_cc310_mbedcrypto_${NRF_CC310_VER}.a
  )
  if(NOT EXISTS ${NRF_CC310_LIB})
    message(WARNING "This combination of SoC, floating point ABI, and interrupts settings is not supported by the nrf_cc310_mbedcryto lib."
                    "(${NRF_CC310_LIB} doesn't exist.)")
  endif()
  add_library(${IMAGE}mbedcrypto_cc310 STATIC IMPORTED GLOBAL)
  set_target_properties(${IMAGE}mbedcrypto_cc310 PROPERTIES IMPORTED_LOCATION
                        "${NRF_CC310_LIB}")
  target_include_directories(${IMAGE}mbedcrypto_cc310 INTERFACE
                             "${NRF_CC310_BASE}/include"
                             "${NRF_CC310_BASE}/include/mbedtls")
  target_link_libraries(${IMAGE}mbedcrypto_cc310 INTERFACE ${IMAGE}platform_cc310)
  if (NOT CONFIG_NRF_CRYPTO_BACKEND_COMBINATION_0)
    # Only CC310 backend is selected for mbedtls API.
    # Thus export the alt headers through Zephyr interface library.
    zephyr_include_directories(${NRF_CC310_BASE}/include/mbedtls
                               ${NRF_CC310_BASE}/include)
  endif()
endif()
