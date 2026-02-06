if (NOT PICO_PROGRAM_MAIN_ENTRY)
    # ==========================================
    # CONFIGURATION EVAN SCORPIO (Pins 3 et 4)
    # ==========================================
    set(BOOT_WORKAROUND ON) 

    # --- PISTE 1 (Trou 3) ---
    # Correspond au GPIO 19
    set(OUTPUT_DATA_PIN 19)

    # --- PISTE 2 (Trou 4) ---
    # Automatique : 19 + 1 = 20 (Trou 4)
    set(SECOND_SEGMENT_INDEX 340)

    set(SECOND_SEGMENT_REVERSED OFF)

    # SPI (On laisse par défaut)
    set(OUTPUT_SPI_DATA_PIN 3)
    set(OUTPUT_SPI_CLOCK_PIN 2)
    set(OUTPUT_SPI_INTERFACE spi0)
endif()

cmake_minimum_required(VERSION 3.13)

# Disable Pico 'Reset device'
add_definitions ( -DPICO_STDIO_USB_ENABLE_RESET_VIA_VENDOR_INTERFACE=0 )

# initialize the SDK based on PICO_SDK_PATH
set(PICO_SDK_PATH ${CMAKE_CURRENT_SOURCE_DIR}/sdk/pico)
set(FREERTOS_KERNEL_PATH ${CMAKE_CURRENT_SOURCE_DIR}/sdk/freertos)

include(${PICO_SDK_PATH}/external/pico_sdk_import.cmake)
include(${FREERTOS_KERNEL_PATH}/portable/ThirdParty/GCC/RP2040/FreeRTOS_Kernel_import.cmake)

project(HyperSerialPico  C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

IF(CMAKE_COMPILER_IS_GNUCC)
    string(REGEX REPLACE "(\-O[011123456789])" "" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
    string(REGEX REPLACE "(\-O[011123456789])" "" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Og")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -Og")
ENDIF(CMAKE_COMPILER_IS_GNUCC)

pico_sdk_init()

set(HyperSerialPicoCompanionLibs FreeRTOS-Kernel FreeRTOS-Kernel-Heap1 pico_stdlib pico_multicore hardware_pio hardware_dma hardware_spi)
set(HyperSerialPicoCompanionIncludes ${CMAKE_CURRENT_SOURCE_DIR}/include ${CMAKE_CURRENT_SOURCE_DIR}/sdk/config)
file(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/generated)
file(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/firmware)

if(NOT CMAKE_HOST_WIN32)
	string(ASCII 27 EscChar)
	set(ColorReset  "${EscChar}[m")
	set(GreenColor  "${EscChar}[32m")
	set(YellowColor "${EscChar}[33m")
endif()

# Overrides via command line
if (OVERRIDE_DATA_PIN)
	set(OUTPUT_DATA_PIN ${OVERRIDE_DATA_PIN})
endif()
if (OVERRIDE_BOOT_WORKAROUND)
	set(BOOT_WORKAROUND ${OVERRIDE_BOOT_WORKAROUND})
endif()

# NETTOYAGE DES LOGS ET AFFICHAGE REEL
message( STATUS "---------------------------")
message( STATUS "Config Evan SCORPIO (1+2 et 3) :")
message( STATUS "Piste 1 (DATA_PIN) : GPIO ${OUTPUT_DATA_PIN} (LEDs 0-339)")
message( STATUS "Piste 2 (SEGMENT_2): GPIO 17 (LEDs 340+)")
message( STATUS "Boot workaround    : ${BOOT_WORKAROUND}")
message( STATUS "---------------------------")

add_compile_options(-ftrack-macro-expansion=0 -fno-diagnostics-show-caret -fdiagnostics-color=auto)

macro(HyperSerialPicoTarget HyperSerialPicoTargetName)
    if (PICO_PROGRAM_MAIN_ENTRY)
        add_executable(${HyperSerialPicoTargetName} ${PICO_PROGRAM_MAIN_ENTRY})
    else()
        add_executable(${HyperSerialPicoTargetName} ${CMAKE_CURRENT_SOURCE_DIR}/source/main.cpp)
    endif()
    if (BOOT_WORKAROUND)
        target_compile_definitions(${HyperSerialPicoTargetName} PUBLIC -DBOOT_WORKAROUND -DPICO_XOSC_STARTUP_DELAY_MULTIPLIER=64)
    endif()
    target_include_directories(${HyperSerialPicoTargetName} PRIVATE ${HyperSerialPicoCompanionIncludes})
    target_link_libraries(${HyperSerialPicoTargetName} ${HyperSerialPicoCompanionLibs})
    pico_add_extra_outputs(${HyperSerialPicoTargetName})
    pico_enable_stdio_usb(${HyperSerialPicoTargetName} 1)
    pico_enable_stdio_uart(${HyperSerialPicoTargetName} 0)
    pico_generate_pio_header(${HyperSerialPicoTargetName} ${CMAKE_CURRENT_SOURCE_DIR}/pio/neopixel.pio OUTPUT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/generated)
    pico_generate_pio_header(${HyperSerialPicoTargetName} ${CMAKE_CURRENT_SOURCE_DIR}/pio/neopixel_ws2812b.pio OUTPUT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/generated)
    add_custom_command(TARGET ${HyperSerialPicoTargetName} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/${HyperSerialPicoTargetName}.uf2 ${CMAKE_CURRENT_SOURCE_DIR}/firmware)
endmacro()

IF(NOT SECOND_SEGMENT_INDEX)
    # Single segment logic (Ignoré)
ELSE()
    IF(NOT SECOND_SEGMENT_REVERSED)
        # Compilation du firmware SK6812 Neutral White
        HyperSerialPicoTarget("${CMAKE_PROJECT_NAME}_sk6812Neutral_multisegment_at_${SECOND_SEGMENT_INDEX}")
        target_compile_definitions("${CMAKE_PROJECT_NAME}_sk6812Neutral_multisegment_at_${SECOND_SEGMENT_INDEX}" PRIVATE -DNEOPIXEL_RGBW -DDATA_PIN=${OUTPUT_DATA_PIN} -DSECOND_SEGMENT_START_INDEX=${SECOND_SEGMENT_INDEX})
    ELSE()
        # Reversed logic (Ignoré)
    ENDIF()
ENDIF()
