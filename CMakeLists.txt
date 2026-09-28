# SPDX-License-Identifier: MIT

cmake_minimum_required(VERSION 3.13)

include(pico-sdk/pico_sdk_init.cmake)

project(RB-2040-uart-bridge)

if (NOT PICO_BOARD)
	set(PICO_BOARD pico CACHE STRING "Target board")
endif()

pico_sdk_init()

# Build one firmware variant for a given UART0 pin pair.
#   target   CMake target name
#   tx / rx  UART0 GPIO numbers
#   outname  base name of the generated artifacts (.uf2/.elf/...)
function(add_bridge_target target tx rx outname)
	add_executable(${target} uart-bridge.c usb-descriptors.c)

	target_include_directories(${target} PUBLIC
		./
		pico-sdk/lib/tinyusb/src)

	target_compile_definitions(${target} PRIVATE
		UART0_TX_PIN=${tx}
		UART0_RX_PIN=${rx})

	target_link_libraries(${target}
		hardware_flash
		pico_multicore
		pico_stdlib
		tinyusb_device)

	set_target_properties(${target} PROPERTIES OUTPUT_NAME ${outname})

	pico_add_extra_outputs(${target})
endfunction()

# UART0 on GP12/GP13 - current RB-2040 board.
add_bridge_target(RB-2040-uart-bridge-gp12 12 13 RB-2040-uart-bridge-GP12-13)

# UART0 on GP16/GP17 - older module.
add_bridge_target(RB-2040-uart-bridge-gp16 16 17 RB-2040-uart-bridge-GP16-17)
