# gnu makefile to build arm program to load ice40 bitstreams
#   using STM32Cube library
#
# make elf	- build executable (can be loaded by openocd)
# make hex	- build intelhex file (can be loaded by stm32flash)
# make flash	- build and load with stm32flash
# make oflash	- build and load with openocd
# make dfu      - build raw image and load with dfu-util

TARGET	= iceboot
SRC	= iceboot.c
OCDCONFIG = stlink.cfg

# default target
all: elf

# select STM32 SoC
#MCU	= STM32L476xx
#ASM	= startup_stm32l476xx.s
#LDSCRIPT = ../Src/sdk_flash.ld
MCU	= STM32L433xx
ASM	= startup_stm32l433xx.s
LDSCRIPT = ../Src/blackice_flash.ld

# select ARM architecture
ARCH	= -mcpu=cortex-m3
#ARCH	= -mcpu=cortex-m4

# select floating-point support (cortex-m4 only)
#FPU	= -mfloat-abi=softfp -mfpu=fpv4-sp-d16
#FPU	= -mfloat-abi=hard -mfpu=fpv4-sp-d16

# uncomment to include debug symbols
DEBUG	= -g

# directories
#OUTPUT_DIR	= $(TARGET)_output
OUTPUT_DIR	= output
OBJ_DIR		= $(OUTPUT_DIR)/obj

DRIVERS		= ../Drivers/STM32L4xx_HAL_Driver
USB			= ../Middlewares/ST/STM32_USB_Device_Library
CMSIS		= ../Drivers/CMSIS/Device/ST/STM32L4xx

INCLUDE	= \
	-I$(USB)/Core/Inc \
	-I$(USB)/Class/CDC/Inc \
	-I../Inc \
	-I../Drivers/CMSIS/Include \
	-I$(DRIVERS)/Inc \
	-I$(CMSIS)/Include \

VPATH	= \
	$(DRIVERS)/Src \
	$(USB)/Core/Src \
	$(USB)/Class/CDC/Src \
	$(CMSIS)/Source/Templates/gcc \
	$(CMSIS)/Source/Templates

# source and object files
SRC +=  \
	main.c \
	usbd_cdc_if.c \
	usbd_conf.c \
	usbd_desc.c \
	usb_device.c \
	stm32l4xx_it.c \
	stm32l4xx_hal_msp.c \
	system_stm32l4xx.c

LIB_SRC := $(wildcard $(DRIVERS)/Src/*.c) $(wildcard $(USB)/Core/Src/*.c) $(wildcard $(USB)/Class/CDC/Src/*.c)

LIB_OBJECTS	:= $(addprefix $(OBJ_DIR)/,$(notdir $(LIB_SRC:.c=.o)))
ASM_OBJECTS	:= $(addprefix $(OBJ_DIR)/,$(notdir $(ASM:.s=.o)))
SRC_OBJECTS	:= $(addprefix $(OBJ_DIR)/,$(notdir $(SRC:.c=.o)))

LIB	= $(OUTPUT_DIR)/drivers.a

# toolchain definitions
CC	= arm-none-eabi-gcc
AR	= arm-none-eabi-ar

CPPFLAGS	= -D$(MCU) $(INCLUDE)

CFLAGS	= -ffunction-sections -fdata-sections -O0 -Werror --std=c11 -Wall
CFLAGS	+= -mthumb $(ARCH) $(FPU) $(DEBUG)

LDFLAGS	= -Wl,--gc-sections -Wall
#LDFLAGS += -Wl,-Map=$(OUTPUT_DIR)/$(TARGET).map,--cref
LDFLAGS += -mthumb $(ARCH) $(FPU) $(DEBUG)

# toolchain recipes
$(OBJ_DIR)/%.o: %.c
	$(CC) -c $(CPPFLAGS) $(CFLAGS) $< -o $@

$(OBJ_DIR)/%.o: %.s
	$(CC) -c $(CPPFLAGS) $(CFLAGS) $< -o $@

PAD = $(OUTPUT_DIR)/$(TARGET).pad
$(OUTPUT_DIR)/$(TARGET).elf : $(LIB) $(ASM_OBJECTS) $(SRC_OBJECTS) $(LDSCRIPT)
	$(CC) $(LDFLAGS) $(ASM_OBJECTS) $(SRC_OBJECTS) $(LIB) -T$(LDSCRIPT) -o $@
	arm-none-eabi-size $@ | tail -1 | awk '{printf "0x%x\n",int((0x08000000+$$1+$$2+7)/8)*8}' > $(PAD)

# dependency rules

HEADERS := $(wildcard ../Inc/*.h \
	../Drivers/CMSIS/Include/*.h \
	../Drivers/CMSIS/Device/ST/STM32L4xx/Include/*.h \
	../Drivers/STM32L4xx_HAL_Driver/Inc/*.h \
	../Drivers/STM32L4xx_HAL_Driver/Inc/legacy/*.h)

$(SRC_OBJECTS) $(LIB_OBJECTS): $(HEADERS)
$(ASM_OBJECTS) $(SRC_OBJECTS) $(LIB_OBJECTS): | $(OBJ_DIR)

$(OBJ_DIR):
	-mkdir -p $(OBJ_DIR)

$(LIB): $(LIB_OBJECTS)
	-rm -f $@
	$(AR) crv $@ $^

.INTERMEDIATE: $(LIB_OBJECTS)

$(OUTPUT_DIR)/$(TARGET).hex : $(OUTPUT_DIR)/$(TARGET).elf
	arm-none-eabi-objcopy -O ihex --pad-to `cat $(PAD)` $< $@

$(OUTPUT_DIR)/$(TARGET).raw: $(OUTPUT_DIR)/$(TARGET).elf
	arm-none-eabi-objcopy -O binary --pad-to `cat $(PAD)` $< $@

# top-level targets
clean:
	-rm -r $(OUTPUT_DIR)

elf: $(OUTPUT_DIR)/$(TARGET).elf

hex: $(OUTPUT_DIR)/$(TARGET).hex

raw: $(OUTPUT_DIR)/$(TARGET).raw

flash: hex
	stm32flash -w $(OUTPUT_DIR)/$(TARGET).hex -v -g 0x0 /dev/ttyAMA0

oflash: elf
	openocd -f $(OCDCONFIG) -c "program $(OUTPUT_DIR)/$(TARGET).elf verify reset exit"

dfu: raw
	dfu-util -s 0x08000000:leave -a 0 -D $(OUTPUT_DIR)/$(TARGET).raw -t 1024
