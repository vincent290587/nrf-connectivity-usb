cmake_minimum_required(VERSION 3.15.2)

project(Connectivity LANGUAGES ASM C CXX)

set(APP_VERSION 1)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_VERBOSE_MAKEFILE        OFF)
set(CMAKE_COLOR_MAKEFILE          OFF)

set(USE_SOFTDEVICE                ON)

set(USE_CONNECTIVITY              ON)

set(SDK_ROOT    "c:/Nordic/nRF5_SDK_16.0.0_98a08e2")
set(PROJ_DIR    "${CMAKE_CURRENT_SOURCE_DIR}")

if (CMAKE_BUILD_TYPE STREQUAL "Release")
    add_definitions(-DNRF_LOG_ENABLED=0)
    message(STATUS "NRF_LOG_ENABLED=0")
else()
    message(STATUS "NRF_LOG_ENABLED=1")
    add_definitions(-DNRF_LOG_ENABLED=1)
    add_definitions(-DNRF_LOG_DEFERRED=0)
    add_definitions(-DNRF_LOG_BACKEND_RTT_ENABLED=1)
    add_definitions(-DNRF_LOG_BACKEND_UART_ENABLED=0)
#    add_definitions(-DNRFX_WDT_ENABLED=0)
    add_definitions(-DDEBUG -DDEBUG_NRF -DDEBUG_NRF_USER)
endif()

add_definitions(-DBOARD_PCA10056)

add_definitions(-DSER_PHY_HCI)
add_definitions(-DSER_CONNECTIVITY)
add_definitions(-DSER_PHY_HCI_USB_CDC)
add_definitions(-DBLE_STACK_SUPPORT_REQD)
add_definitions(-DANT_STACK_SUPPORT_REQD)
add_definitions(-DBYPASS_CONN_ERROR_HANDLING)
add_definitions(-DS340)
add_definitions(-D__HEAP_SIZE=8192)
add_definitions(-D__STACK_SIZE=8192)

add_definitions(-DPNP_ID_PRODUCT_VERSION=${APP_VERSION})
add_definitions(-DANT_LICENSE_KEY="3831-521d-7df9-24d8-eff3-467b-225f-a00e")
add_definitions(-DUSE_SVIEW=0 -DUSE_RTT=1) # we need NRF_LOG_ENABLED=1
#add_definitions(-DUSE_SVIEW=0 -DUSE_JSCOPE -DUSE_RTT=1) # we need NRF_LOG_ENABLED=0
#add_definitions(-DUSE_APP_CONFIG)
add_definitions(-DNRF52840_XXAA)
add_definitions(-DCONFIG_NFCT_PINS_AS_GPIOS)
add_definitions(-DCONFIG_GPIO_AS_PINRESET)
add_definitions(-DNRF_SD_BLE_API_VERSION=7)
#add_definitions(-DNRFX_SAADC_API_V2)
#add_definitions(-DAPP_TIMER_V2)
#add_definitions(-DAPP_TIMER_V2_RTC1_ENABLED)
add_definitions(-DAPP_TIMER_KEEPS_RTC_ACTIVE)
add_definitions(-DSWI_DISABLE0)
add_definitions(-DNRF52_PAN_74)
add_definitions(-DNRF_PWR_MGMT_CONFIG_FPU_SUPPORT_ENABLED=1)
add_definitions(-DFLOAT_ABI_HARD -fsingle-precision-constant)
add_definitions(-Wdouble-promotion -Wfloat-conversion -Werror=return-type)
add_definitions(-Wno-attributes -Wsign-compare -Wenum-compare -Wno-error=deprecated)
add_definitions(-Werror)


add_link_options(-L${SDK_ROOT}/modules/nrfx/mdk)
add_link_options(-T${CMAKE_CURRENT_SOURCE_DIR}/ble_connectivity_gcc_nrf52.ld)
add_link_options(-mfloat-abi=hard -mfpu=fpv4-sp-d16)
add_link_options(-Wl,--gc-sections)
add_link_options(--specs=nano.specs)

add_compile_options(-DARM_MATH_CM4)
add_compile_options(-mfloat-abi=hard -mfpu=fpv4-sp-d16)
add_compile_options(-ffunction-sections -fdata-sections -fno-strict-aliasing)
add_compile_options(-fno-builtin -fshort-enums -nostdlib -fno-exceptions)

set(CMAKE_CXX_FLAGS "-std=gnu++0x -felide-constructors -fno-exceptions -fno-rtti" CACHE STRING "")

# Includes used by the SDK only
include_directories(
        ${PROJ_DIR}/.
        ${PROJ_DIR}/config
    )

add_subdirectory(sdk    ./libs/sdk/)

# if a file is added in one of those dirs, the cmake... command must be re-run
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR} SRCS)

add_executable(${PROJECT_NAME}.elf ${SRCS})

target_link_libraries(${PROJECT_NAME}.elf nosys)
target_link_libraries(${PROJECT_NAME}.elf -Wl,--whole-archive SDK_LIBS -Wl,--no-whole-archive)

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND ${TOOLCHAIN}-objcopy ARGS -O binary ${PROJECT_NAME}.elf ${PROJECT_NAME}.bin)

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND ${TOOLCHAIN}-objcopy ARGS -O ihex ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex)

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND size --format=berkeley "${PROJECT_NAME}.elf"
        COMMENT "Invoking: Print Size")

#add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
#        COMMAND ${NRFUTIL} settings generate --family NRF52 --application-version ${APP_VERSION} --application ${PROJECT_NAME}.hex --bootloader-version 1 --bl-settings-version 2 settings_dfu.hex
#        COMMAND ${MERGEHEX} --merge ${PROJECT_NAME}.hex settings_dfu.hex --output app_dfu_settings.hex
#        COMMAND ${MERGEHEX} --merge app_dfu_settings.hex ${CMAKE_CURRENT_SOURCE_DIR}/ANT_s332_nrf52_7.0.1.hex --output sd_app_dfu_settings.hex
#        COMMENT "Merging full app...")

find_program(NRFUTIL "nrfutil")
find_program(NRFJPROG "nrfjprog")
find_program(MERGEHEX "mergehex" HINTS "c:/Program Files (x86)/Nordic Semiconductor/nrf5x/bin")

if(NOT NRFJPROG)
    message(FATAL_ERROR "nrfjprog not found, not able to merge connectivity firmware with SoftDevice.")
endif()

if(NOT MERGEHEX)
    message(FATAL_ERROR "mergehex not found, not able to merge connectivity firmware with SoftDevice.")
endif()

if(NOT NRFUTIL)
    message(FATAL_ERROR "nrfutil not found, not able to create DFU packages of connectivity firmware.")
endif()

add_custom_target(flash
        DEPENDS ${PROJECT_NAME}.hex
        COMMAND ${NRFJPROG} --program ${PROJECT_NAME}.hex -f nrf52 --sectorerase
        COMMAND ${NRFJPROG} --reset -f nrf52
        )

add_custom_target(flash_sd
        COMMAND ${NRFJPROG} --program ${SDK_ROOT}/components/softdevice/s340/hex/ANT_s340_nrf52_7.0.1.hex -f nrf52 --sectorerase
        )

