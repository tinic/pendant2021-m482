cmake_minimum_required(VERSION 3.10)

# M48x M480 M482 M482ZGCAE

if(BOOTLOADER)
    set(BASE_ADDRESS 0x00000000)
    project(pendant2021-bootloader C CXX ASM)
    set(MAINCPU_DEFINITIONS BOOTLOADER)
    set(MAINMCU_LD_SCRIPT ${PROJECT_SOURCE_DIR}/gcc_arm_128k_bootloader.ld)
endif(BOOTLOADER)
    
if(BOOTLOADED)
    set(BASE_ADDRESS 0x0000C000)
    project(pendant2021-bootloaded C CXX ASM)
    set(MAINCPU_DEFINITIONS BOOTLOADED)
    set(MAINMCU_LD_SCRIPT ${PROJECT_SOURCE_DIR}/gcc_arm_128k_bootloaded.ld)
endif(BOOTLOADED)

if(TESTING)
    set(BASE_ADDRESS 0x00000000)
    project(pendant2021-testing C CXX ASM)
    set(MAINCPU_DEFINITIONS BOOTLOADED TESTING)
    set(MAINMCU_LD_SCRIPT ${PROJECT_SOURCE_DIR}/gcc_arm_128k_vscode.ld)
endif(TESTING)

set(CC_FLAGS
    -flto
    -mcpu=cortex-m4 
    -Wnull-dereference
    -mthumb
    -mfloat-abi=hard
    -mfpu=fpv4-sp-d16
    -fdata-sections
    -ffunction-sections
    -Wall
    -Wextra
    -Wno-sign-compare
    -Wno-unused-parameter
    -Wno-strict-aliasing 
    -Wno-format
    -ffast-math
    -fallow-store-data-races)

set(CXX_FLAGS 
    -Wall 
    -Wextra
    -Wdouble-promotion
    -Wshadow 
    -Wfloat-conversion 
    -Wno-volatile
    -Wformat=2 
    -std=c++20 
    -fno-rtti 
    -fno-exceptions)

set(LD_FLAGS
    -T${MAINMCU_LD_SCRIPT}
    -flto
    -mcpu=cortex-m4
    -mthumb
    -mfloat-abi=hard
    -mfpu=fpv4-sp-d16
    -specs=nano.specs
    -u _printf_float
    -lc
    -lm
    -lnosys
    -Wl,-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map,--cref
    -Wl,--gc-sections)

set(DEBUG_FLAGS
    -Og -g)

set(RELEASE_FLAGS   
    -Os)

set(DEFINITIONS NDEBUG ARM_MATH_CM4 __FPU_PRESENT ${MAINCPU_DEFINITIONS})

set(HEX_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin)
set(DUMP_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.S)

file(GLOB_RECURSE USB_CORE_SRC ${PROJECT_SOURCE_DIR}/Library/UsbHostLib/src_core/*.c)
file(GLOB_RECURSE NUVOTON0_SRC ${PROJECT_SOURCE_DIR}/Library/Device/Nuvoton/GCC/*.c)
file(GLOB_RECURSE NUVOTON1_SRC ${PROJECT_SOURCE_DIR}/Library/Device/Nuvoton/M480/Source/*.c)

add_executable(${PROJECT_NAME}.elf 
    ${PROJECT_SOURCE_DIR}/i2cmanager.cpp
    ${PROJECT_SOURCE_DIR}/leds.cpp
    ${PROJECT_SOURCE_DIR}/color.cpp
    ${PROJECT_SOURCE_DIR}/model.cpp
    ${PROJECT_SOURCE_DIR}/timeline.cpp
    ${PROJECT_SOURCE_DIR}/pendant.cpp
    ${PROJECT_SOURCE_DIR}/bootloader.cpp
    ${PROJECT_SOURCE_DIR}/sdcard.cpp
    ${PROJECT_SOURCE_DIR}/input.cpp
    ${PROJECT_SOURCE_DIR}/stm32wl.cpp
    ${PROJECT_SOURCE_DIR}/sdd1306.cpp
    ${PROJECT_SOURCE_DIR}/effects.cpp
    ${PROJECT_SOURCE_DIR}/ui.cpp
    ${PROJECT_SOURCE_DIR}/seed.cpp
    ${PROJECT_SOURCE_DIR}/stubs.c
    ${PROJECT_SOURCE_DIR}/descriptors.c
    ${PROJECT_SOURCE_DIR}/msc.cpp
    ${PROJECT_SOURCE_DIR}/fatfs/ff.c
    ${PROJECT_SOURCE_DIR}/fatfs/ffsystem.c
    ${PROJECT_SOURCE_DIR}/fatfs/ffunicode.c
    ${PROJECT_SOURCE_DIR}/main.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/retarget.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/clk.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/fmc.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/gpio.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/i2c.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/pdma.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/spi.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/sys.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/timer.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/uart.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/epwm.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/usbd.c
    ${PROJECT_SOURCE_DIR}/Library/StdDriver/src/qspi.c
    ${USB_CORE_SRC}
    ${NUVOTON0_SRC}
    ${NUVOTON1_SRC}
    ${PROJECT_SOURCE_DIR}/Library/Device/Nuvoton/M480/Source/GCC/startup_M480.S)

target_include_directories(${PROJECT_NAME}.elf PRIVATE
    .
    fatfs
    Library/CMSIS/Include
    Library/Device/Nuvoton/M480/Include
    Library/UsbHostLib/inc
    Library/UsbHostLib/src_msc
    Library/UsbHostLib/src_uac
    Library/StdDriver/inc)

# Generate version.h
find_package(Git)
if(GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
    execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} OUTPUT_VARIABLE GIT_SHORT_SHA OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(COMMAND ${GIT_EXECUTABLE} rev-list HEAD --count WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} OUTPUT_VARIABLE GIT_REV_COUNT OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(COMMAND ${GIT_EXECUTABLE} show -s --format=%ad HEAD --date=iso-strict WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} OUTPUT_VARIABLE GIT_COMMIT_DATE OUTPUT_STRIP_TRAILING_WHITESPACE)
else()
    set(GIT_SHORT_SHA "unknown")
    set(GIT_REV_COUNT "unknown")
    set(GIT_COMMIT_DATE "unknown")
endif()

target_include_directories(${PROJECT_NAME}.elf PRIVATE ${CMAKE_BINARY_DIR})
configure_file("${PROJECT_SOURCE_DIR}/version.h.in" "${CMAKE_BINARY_DIR}/version.h" @ONLY)

# Generate font.h
if(${CMAKE_HOST_SYSTEM_NAME} MATCHES "Windows")
    execute_process(COMMAND python3 ${PROJECT_SOURCE_DIR}/font_convert.py -i ${PROJECT_SOURCE_DIR}/font.gif -o ${CMAKE_BINARY_DIR}/font.h -v font_data)
else(${CMAKE_HOST_SYSTEM_NAME} MATCHES "Windows")
    execute_process(COMMAND ${PROJECT_SOURCE_DIR}/font_convert.py -i ${PROJECT_SOURCE_DIR}/font.gif -o ${CMAKE_BINARY_DIR}/font.h -v font_data)
endif(${CMAKE_HOST_SYSTEM_NAME} MATCHES "Windows")

if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    message(FATAL_ERROR "Compiler must be GCC.")
endif(NOT CMAKE_CXX_COMPILER_ID STREQUAL "GNU")

if(CMAKE_CXX_COMPILER_VERSION LESS "10.2")
    message(FATAL_ERROR "GCC must be version 10.2 or higher (found version '" ${CMAKE_CXX_COMPILER_VERSION} "')")
endif(CMAKE_CXX_COMPILER_VERSION LESS "10.2")

target_compile_definitions(${PROJECT_NAME}.elf PRIVATE ${DEFINITIONS})
target_compile_options(${PROJECT_NAME}.elf PRIVATE ${CC_FLAGS})
target_link_options(${PROJECT_NAME}.elf PRIVATE ${LD_FLAGS})
target_compile_options(${PROJECT_NAME}.elf PRIVATE "$<$<COMPILE_LANGUAGE:CXX>:${CXX_FLAGS}>")
target_compile_options(${PROJECT_NAME}.elf PRIVATE "$<$<CONFIG:DEBUG>:${DEBUG_FLAGS}>")
target_compile_options(${PROJECT_NAME}.elf PRIVATE "$<$<CONFIG:RELEASE>:${RELEASE_FLAGS}>")

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJDUMP} -D $<TARGET_FILE:${PROJECT_NAME}.elf> > ${DUMP_FILE}
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE}
    COMMAND ${CMAKE_OBJCOPY} -O ihex -R .eeprom -R .fuse -R .lock -R .signature $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE}
    COMMAND ${CMAKE_SIZE} ${PROJECT_NAME}.elf
    COMMENT "Building ${HEX_FILE} \nBuilding ${BIN_FILE}")

set(PROGRAM_CMD "./openocd -f ./stlink.cfg -f ./numicroM4.cfg -c \"program ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin ${BASE_ADDRESS} verify reset exit\"")
install(CODE "execute_process(COMMAND ${PROGRAM_CMD} WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/openocd)")
