# This file is part of the MicroPython project, http://micropython.org/
#
# The MIT License (MIT)
#
# Copyright 2019 Sony Semiconductor Solutions Corporation
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.

# Select the board to build for.
ifeq ($(BOARD),)
  $(error You must provide a BOARD parameter)
else
  ifeq ($(wildcard boards/$(BOARD)/.),)
    $(error Invalid BOARD specified)
  endif
endif

# If the build directory is not given, make it reflect the board name.
BUILD ?= build-$(BOARD)

include ../../py/mkenv.mk

# Board-specific
include boards/$(BOARD)/mpconfigboard.mk

# Port-specific
include mpconfigport.mk

# CircuitPython-specific
include $(TOP)/py/circuitpy_mpconfig.mk

# qstr definitions (must come before including py.mk)
QSTR_DEFS = qstrdefsport.h

# include py core make definitions
include $(TOP)/py/py.mk

include $(TOP)/supervisor/supervisor.mk

# Include make rules and variables common across CircuitPython builds.
include $(TOP)/py/circuitpy_defns.mk

CROSS_COMPILE = arm-none-eabi-

SPRESENSE_SDK = spresense-exported-sdk

FIRMWARE = $(SPRESENSE_SDK)/firmware

# Platforms are: Linux, Darwin, MSYS, CYGWIN
PLATFORM := $(firstword $(subst _, ,$(shell uname -s 2>/dev/null)))

ifeq ($(PLATFORM),Darwin)
  # macOS
  MKSPK = mkspk/mkspk
else ifeq ($(PLATFORM),Linux)
  # Linux
  MKSPK = mkspk/mkspk
else
  # Cygwin/MSYS2
  MKSPK = mkspk/mkspk.exe
endif

SERIAL ?= /dev/ttyUSB0

INC += \
	-I. \
	-I../.. \
	-I../lib/mp-readline \
	-I../lib/timeutils \
	-I../../lib/tinyusb/src \
	-I../../supervisor/shared/usb \
	-Iboards/$(BOARD) \
	-I$(BUILD) \
	-I$(SPRESENSE_SDK)/nuttx/include \
	-I$(SPRESENSE_SDK)/nuttx/arch \
	-I$(SPRESENSE_SDK)/nuttx/arch/chip \
	-I$(SPRESENSE_SDK)/nuttx/arch/os \
	-I$(SPRESENSE_SDK)/sdk/bsp/include \
	-I$(SPRESENSE_SDK)/sdk/bsp/include/sdk \

CFLAGS += \
	$(INC) \
	-DCONFIG_WCHAR_BUILTIN \
	-DCONFIG_HAVE_DOUBLE \
	-Dmain=spresense_main \
	-D_estack=__stack \
	-c \
	-Os \
	-pipe \
	-std=gnu11 \
	-mcpu=cortex-m4 \
	-mthumb \
	-mfpu=fpv4-sp-d16 \
	-mfloat-abi=hard \
	-mabi=aapcs \
	-fno-builtin \
	-fno-strict-aliasing \
	-fno-strength-reduce \
	-fomit-frame-pointer \
	-ffunction-sections \
	-fdata-sections \
	-Wall \

LIBM = "${shell "$(CC)" $(CFLAGS) -print-file-name=libm.a}"

LIBGCC = "${shell "$(CC)" $(CFLAGS) -print-libgcc-file-name}"

LDFLAGS = \
	--entry=__start \
	-nostartfiles \
	-nodefaultlibs \
	-T$(SPRESENSE_SDK)/nuttx/build/ramconfig.ld \
	--gc-sections \
	-Map=$(BUILD)/output.map \
	-o $(BUILD)/firmware.elf \
	--start-group \
	-u spresense_main \
	-u board_timerhook \
	$(BUILD)/libmpy.a \
	$(SPRESENSE_SDK)/sdk/libs/libapps.a \
	$(SPRESENSE_SDK)/sdk/libs/libsdk.a \
	$(LIBM) \
	$(LIBGCC) \
	--end-group \
	-L$(BUILD) \

CFLAGS += -DCFG_TUSB_MCU=OPT_MCU_CXD56 -DCFG_TUD_MIDI_RX_BUFSIZE=128 -DCFG_TUD_CDC_RX_BUFSIZE=256 -DCFG_TUD_MIDI_TX_BUFSIZE=128 -DCFG_TUD_CDC_TX_BUFSIZE=256 -DCFG_TUD_MSC_BUFSIZE=512 $(CFLAGS_MOD)

SRC_COMMON_HAL_EXPANDED = $(addprefix shared-bindings/, $(SRC_COMMON_HAL)) \
                          $(addprefix shared-bindings/, $(SRC_BINDINGS_ENUMS)) \
                          $(addprefix common-hal/, $(SRC_COMMON_HAL))

SRC_SHARED_MODULE_EXPANDED = $(addprefix shared-bindings/, $(SRC_SHARED_MODULE)) \
                             $(addprefix shared-module/, $(SRC_SHARED_MODULE))

SRC_S = supervisor/cpu.s

SRC_C = \
	tick.c \
	background.c \
	fatfs_port.c \
	mphalport.c \
	boards/$(BOARD)/board.c \
	boards/$(BOARD)/pins.c \
	lib/utils/stdout_helpers.c \
	lib/utils/pyexec.c \
	lib/libc/string0.c \
	lib/mp-readline/readline.c \
	lib/timeutils/timeutils.c \
	lib/oofatfs/ff.c \
	lib/oofatfs/option/ccsbcs.c \
	lib/utils/interrupt_char.c \
	lib/utils/sys_stdio_mphal.c \
	lib/utils/context_manager_helpers.c \
	lib/utils/buffer_helper.c \
	supervisor/shared/memory.c \
	lib/tinyusb/src/portable/sony/cxd56/dcd_cxd56.c \

OBJ = $(PY_O) $(SUPERVISOR_O) $(addprefix $(BUILD)/, $(SRC_C:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(SRC_S:.s=.o))
OBJ += $(addprefix $(BUILD)/, $(SRC_COMMON_HAL_EXPANDED:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(SRC_SHARED_MODULE_EXPANDED:.c=.o))
ifeq ($(INTERNAL_LIBM),1)
OBJ += $(addprefix $(BUILD)/, $(SRC_LIBM:.c=.o))
endif
OBJ += $(addprefix $(BUILD)/, $(SRC_MOD:.c=.o))

# List of sources for qstr extraction
SRC_QSTR += $(SRC_C) $(SRC_SUPERVISOR) $(SRC_COMMON_HAL_EXPANDED) $(SRC_SHARED_MODULE_EXPANDED)
# Sources that only hold QSTRs after pre-processing.
SRC_QSTR_PREPROCESSOR += 

all: $(BUILD)/firmware.spk

$(FIRMWARE):
	$(ECHO) ""
	$(ECHO) "Download the spresense binaries zip archive from:"
	$(ECHO) "https://developer.sony.com/file/download/download-spresense-firmware-v1-4-000"
	$(ECHO) "Extract spresense binaries to $(FIRMWARE)"
	$(ECHO) ""
	$(ECHO) "run make flash-bootloader again to flash bootloader."
	exit 1

$(BUILD)/libmpy.a: $(SPRESENSE_SDK) $(OBJ) 
	$(ECHO) "AR $@"
	$(Q)$(AR) rcs $(BUILD)/libmpy.a $(OBJ)

$(BUILD)/firmware.elf: $(BUILD)/libmpy.a
	$(ECHO) "LD $@"
	$(Q)$(LD) $(LDFLAGS)

$(MKSPK):
	$(MAKE) -C mkspk

$(BUILD)/firmware.spk: $(BUILD)/firmware.elf $(MKSPK)
	$(ECHO) "Creating $@"
	$(MKSPK) -c 2 $(BUILD)/firmware.elf nuttx $(BUILD)/firmware.spk

flash: $(BUILD)/firmware.spk
	$(ECHO) "Writing $< to the board"
	$(SPRESENSE_SDK)/sdk/tools/flash.sh -c $(SERIAL) $(BUILD)/firmware.spk

flash-bootloader: $(SPRESENSE_SDK) $(FIRMWARE)
	$(ECHO) "Writing loader to the board"
	$(SPRESENSE_SDK)/sdk/tools/flash.sh -l $(FIRMWARE) -c $(SERIAL)

include $(TOP)/py/mkrules.mk

# Print out the value of a make variable.
# https://stackoverflow.com/questions/16467718/how-to-print-out-a-variable-in-makefile
print-%:
	@echo $* = $($*)
