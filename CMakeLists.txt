cmake_minimum_required(VERSION 3.13)

if (NOT DEFINED BUILD_TIME_PROVIDED_VERSION)
    # No version passed in, so derive one from git. A commit that is exactly a
    # tag is a release (R); anything ahead of the last tag is a dev build (U).
    # Falls back to 255.255.255 U outside a repo or before the first tag.
    set(BUILD_TIME_PROVIDED_VERSION 255.255.255)
    set(BUILD_VERSION_TYPE U)

    find_package(Git QUIET)
    if (GIT_FOUND)
        execute_process(COMMAND ${GIT_EXECUTABLE} describe --tags --exact-match
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            OUTPUT_VARIABLE GIT_EXACT_TAG OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET RESULT_VARIABLE GIT_EXACT_RESULT)
        execute_process(COMMAND ${GIT_EXECUTABLE} describe --tags --abbrev=0
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            OUTPUT_VARIABLE GIT_LAST_TAG OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET RESULT_VARIABLE GIT_LAST_RESULT)

        if (GIT_EXACT_RESULT EQUAL 0)
            set(GIT_DERIVED_VERSION "${GIT_EXACT_TAG}")
            set(BUILD_VERSION_TYPE R)
        elseif (GIT_LAST_RESULT EQUAL 0)
            set(GIT_DERIVED_VERSION "${GIT_LAST_TAG}")
        endif()

        # project(VERSION) only accepts a numeric major.minor.patch, so pull
        # that out of tags like "1.2.3" or "v1.2.3-rc1".
        if (DEFINED GIT_DERIVED_VERSION)
            string(REGEX MATCH "[0-9]+\\.[0-9]+\\.[0-9]+"
                   GIT_NUMERIC_VERSION "${GIT_DERIVED_VERSION}")
            if (GIT_NUMERIC_VERSION)
                set(BUILD_TIME_PROVIDED_VERSION ${GIT_NUMERIC_VERSION})
            endif()
        endif()
    endif()
else ()
    set(BUILD_VERSION_TYPE R)
endif()
# Release code names, keyed on major.minor. Add a line per named release;
# anything not listed builds as "unnamed".
set(BUILD_NAMES
    "1.0:Pico"
    "1.1:Latte"
    "1.2:Wasp"
)

string(REGEX MATCH "^[0-9]+\\.[0-9]+" BUILD_VERSION_SERIES
       "${BUILD_TIME_PROVIDED_VERSION}")
set(BUILD_NAME "unnamed")
foreach(_entry IN LISTS BUILD_NAMES)
    string(REPLACE ":" ";" _parts "${_entry}")
    list(GET _parts 0 _series)
    list(GET _parts 1 _name)
    if (_series STREQUAL BUILD_VERSION_SERIES)
        set(BUILD_NAME "${_name}")
    endif()
endforeach()

message("BUILD_TIME_PROVIDED_VERSION=${BUILD_TIME_PROVIDED_VERSION} BUILD_VERSION_TYPE=${BUILD_VERSION_TYPE} BUILD_NAME=${BUILD_NAME}")

string(TIMESTAMP BUILD_TIMESTAMP "%s" UTC)

include($ENV{PICO_SDK_PATH}/external/pico_sdk_import.cmake)

project(rp2040-gb-cartridge C CXX ASM)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

project(${PROJECT_NAME} VERSION ${BUILD_TIME_PROVIDED_VERSION})

configure_file(BuildVersion.h.in BuildVersion.h)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(PICO_BOARD_HEADER_DIRS ${CMAKE_SOURCE_DIR})
set(PICO_BOARD MyBoard)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_HELPERS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/cmakeHelpers)

pico_sdk_init()

add_subdirectory(gb-bootloader)
add_subdirectory(gb-vblankhook)

add_subdirectory(libs/pico-littlefs)
target_compile_definitions(littlefs-lib PUBLIC LFS_NO_MALLOC)

add_subdirectory(libs/git-commit-tracking)

add_executable(${PROJECT_NAME} 
    main.c
    GbDma.c
    GbRtc.c
    mbc.c
    webusb.c
    usb_descriptors.c
    RomStorage.c
    GameBoyHeader.c
    ws2812b_spi.c
    )

file(MAKE_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/generated)

pico_generate_pio_header(${PROJECT_NAME} ${CMAKE_CURRENT_LIST_DIR}/gameboy_bus.pio OUTPUT_DIR ${CMAKE_CURRENT_LIST_DIR}/generated)

target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_CURRENT_LIST_DIR})

pico_enable_stdio_usb(${PROJECT_NAME} 0)
pico_enable_stdio_uart(${PROJECT_NAME} 1)
# pico_set_printf_implementation(${PROJECT_NAME} none)
pico_set_linker_script(${PROJECT_NAME} ${CMAKE_CURRENT_LIST_DIR}/linkerscript.ld)

#Target that removes the CMake variable BUILD_TIMESTAMP from cache.
#this forces CMake to rerun.
add_custom_target(
	clear_cache
	COMMAND ${CMAKE_COMMAND} -U BUILD_TIMESTAMP ${CMAKE_BINARY_DIR}
)

add_dependencies(${PROJECT_NAME}
    clear_cache
    BootloaderGb
    gbSaveGameVBlankHook
    )

target_link_libraries(${PROJECT_NAME} 
    pico_stdlib
    pico_bootsel_via_double_reset
    hardware_dma
    hardware_uart
    hardware_i2c
    hardware_spi
    hardware_gpio
    hardware_pio
    hardware_clocks
    hardware_watchdog
    littlefs-lib
    tinyusb_device
    tinyusb_board
    cmake_git_commit_tracking
    )

pico_add_extra_outputs(${PROJECT_NAME})
