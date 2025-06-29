# == DO NOT EDIT THE FOLLOWING LINES for the Raspberry Pi Pico VS Code Extension to work ==
if(WIN32)
    set(USERHOME $ENV{USERPROFILE})
else()
    set(USERHOME $ENV{HOME})
endif()
set(sdkVersion 2.1.1)
set(toolchainVersion 14_2_Rel1)
set(picotoolVersion 2.1.1)
set(picoVscode ${USERHOME}/.pico-sdk/cmake/pico-vscode.cmake)
if (EXISTS ${picoVscode})
    include(${picoVscode})
endif()
# ====================================================================================
cmake_minimum_required(VERSION 3.13)
set(PICO_BOARD_HEADER_DIRS ${CMAKE_CURRENT_LIST_DIR}/boards )
#set(PICO_PLATFORM "rp2040" CACHE STRING "Chip type")
set(PICO_PLATFORM "rp2350" CACHE STRING "Chip type")
set(PICO_BOARD "murmulator" CACHE STRING "Board type")
#set(PICO_BOARD "murmulator2" CACHE STRING "Board type")
#set(PICO_BOARD "rp2040pizero" CACHE STRING "Board type")

include(pico_sdk_import.cmake)

project(genesis C CXX ASM)
SET(BUILD_NAME "${PROJECT_NAME}")

option(I2S "Enable I2S sound" OFF)
option(I2S_CS4334 "Enable I2S CS4334 sound" OFF)
option(VGA "Enable VGA" OFF)
option(TFT "Enable TFT display" OFF)
option(ILI9341 "Enable TFT ILI9341 display" OFF)
option(HDMI "Enable HDMI display" OFF)
option(TV "Enable TV composite output" OFF)
option(SOFTTV "Enable TV soft composite output" OFF)

set(VGA ON)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}  -funroll-loops  -ffast-math -feliminate-unused-debug-types -ffunction-sections -fdata-sections -O2")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}  -funroll-loops  -ffast-math -feliminate-unused-debug-types -ffunction-sections -fdata-sections -O2")

set(OUTPUT_DIR "${CMAKE_SOURCE_DIR}/bin/${PICO_PLATFORM}/${CMAKE_BUILD_TYPE}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${OUTPUT_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${OUTPUT_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${OUTPUT_DIR}")

pico_sdk_init()

add_subdirectory(drivers/audio)

add_subdirectory(drivers/ps2kbd)
add_subdirectory(drivers/nespad)
add_subdirectory(drivers/fatfs)
add_subdirectory(drivers/sdcard)

add_subdirectory(drivers/vga-nextgen)
add_subdirectory(drivers/st7789)
add_subdirectory(drivers/hdmi)
add_subdirectory(drivers/tv)
add_subdirectory(drivers/tv-software)

add_subdirectory(drivers/graphics)

# INCLUDE FILES THAT SHOULD BE COMPILED:
file(GLOB_RECURSE SRC "src/*.cpp" "src/*.c")

message(STATUS "Add source files:")
foreach (SRC_FILE IN LISTS SRC)
	message(STATUS "${SRC_FILE}")
endforeach ()
message(STATUS "")

add_executable(${PROJECT_NAME} ${SRC})

pico_define_boot_stage2(slower_boot2 ${PICO_DEFAULT_BOOT_STAGE2_FILE})
target_compile_definitions(slower_boot2 PRIVATE PICO_FLASH_SPI_CLKDIV=4)
pico_set_boot_stage2(${PROJECT_NAME} slower_boot2)


#set(FAMILY rp2040)
#set(BOARD pico_sdk)

pico_set_program_name(genesis "Sega MegaDrive by xrip for MURMULATOR devboard")
pico_set_program_version(genesis "develop")

target_include_directories(${PROJECT_NAME} PRIVATE src)

IF(TFT)
	target_link_libraries(${PROJECT_NAME} PRIVATE st7789)
	target_compile_definitions(${PROJECT_NAME} PRIVATE TFT)
	SET(BUILD_NAME "${BUILD_NAME}-TFT")
	IF(ILI9341)
		SET(BUILD_NAME "${BUILD_NAME}-ILI9341")
		target_compile_definitions(${PROJECT_NAME} PRIVATE ILI9341)
	ELSE()
		SET(BUILD_NAME "${BUILD_NAME}-ST7789")
	ENDIF()
ELSEIF(HDMI)
	target_link_libraries(${PROJECT_NAME} PRIVATE hdmi)
	target_compile_definitions(${PROJECT_NAME} PRIVATE HDMI)
	SET(BUILD_NAME "${BUILD_NAME}-HDMI")
ELSEIF(TV)
	target_compile_definitions(${PROJECT_NAME} PRIVATE TV)
	target_link_libraries(${PROJECT_NAME} PRIVATE tv)
	SET(BUILD_NAME "${BUILD_NAME}-TV")
ELSEIF(SOFTTV)
	target_compile_definitions(${PROJECT_NAME} PRIVATE SOFTTV)
	target_link_libraries(${PROJECT_NAME} PRIVATE tv-software)
	SET(BUILD_NAME "${BUILD_NAME}-TV-SOFT")
ELSE()
	target_compile_definitions(${PROJECT_NAME} PRIVATE VGA)
	target_link_libraries(${PROJECT_NAME} PRIVATE vga-nextgen)
	SET(BUILD_NAME "${BUILD_NAME}-VGA")
ENDIF()

IF(NOT I2S)
	target_compile_definitions(${PROJECT_NAME} PRIVATE AUDIO_PWM)
ELSEIF(I2S_CS4334)
	target_compile_definitions(${PROJECT_NAME} PRIVATE I2S_CS4334)
	SET(BUILD_NAME "${BUILD_NAME}-I2S-CS4334")
ELSE()
	SET(BUILD_NAME "${BUILD_NAME}-I2S-TDA1387")
ENDIF()

set_target_properties(${PROJECT_NAME} PROPERTIES OUTPUT_NAME "${BUILD_NAME}")

target_link_libraries(${PROJECT_NAME} PRIVATE
		graphics
		audio

		nespad
		sdcard
		ps2kbd
		fatfs

		pico_runtime
        pico_stdlib
        pico_multicore
        hardware_dma
        hardware_pio
        hardware_i2c
        hardware_interp
        hardware_timer
        hardware_clocks
        hardware_pwm
        hardware_flash

		tinyusb_host
)

target_link_options(${PROJECT_NAME} PRIVATE -Xlinker --print-memory-usage --data-sections)
pico_add_extra_outputs(${PROJECT_NAME})



