#
# Target: base name of build target.
#
TARGET=ArduinoSlaveExtruder

MCU = atmega168
ADMCU = m168
F_CPU = 16000000L

PORT = /dev/ttyUSB1
UPLOAD_RATE = 19200
AVRDUDE_PROGRAMMER = stk500v1

SRCS=	Extruder.cpp \
	EEPROMUtils.cpp \
	Heater.cpp \
	PacketProcessor.cpp \
	Timer1.cpp \
	ThermistorTable.cpp \
	Utils.cpp \
	Variables.cpp

# ARDUINO_HOME should be the top-level path to the Arduino install use
# to do the build.
ifndef ARDUINO_HOME
$(error "ARDUINO_HOME is not defined.")
endif
# Locate the AVR tools.  Attempt to use the natively installed ones 
# first, and fall back on the Arduino install if they're not present.
# You can manually specify the location of the AVR tools by defining
# "AVR_TOOLS_PATH".
ifndef AVR_TOOLS_PATH
AVR_TOOLS_PATH=$(dir $(shell which avr-g++))
ifeq ($(AVR_TOOLS_PATH),)
AVR_TOOLS_PATH=$(ARDUINO_HOME)/hardware/tools/avr/bin
endif
endif

CORE=$(ARDUINO_HOME)/hardware/arduino/cores/arduino
CORE_SRCS=$(filter-out %main.cpp, $(wildcard $(CORE)/*.c) $(wildcard $(CORE)/*.cpp))

OUR_LIBRARIES=SimplePacket
ARD_LIBRARIES=EEPROM

LIB_DIRS=$(foreach lib,$(OUR_LIBRARIES), ../libraries/$(lib)) $(foreach lib,$(ARD_LIBRARIES),$(ARDUINO_HOME)/libraries/$(lib))

LIB_SRCS=$(foreach regex,$(addsuffix /*.c,$(LIB_DIRS)) $(addsuffix /*.cpp,$(LIB_DIRS)),$(wildcard $(regex)))

ALL_SRCS=$(CORE_SRCS) $(LIB_SRCS) $(SRCS)

FORMAT=ihex

OBJECTS_DIR=./build

# Name of this Makefile (used for "make depend").
MAKEFILE = Makefile

# Place -D or -U options here
CDEFS = -DF_CPU=$(F_CPU) $(ADDITIONAL_DEFS)
CXXDEFS = -DF_CPU=$(F_CPU) $(ADDITIONAL_DEFS)

# Place -I options here
CINCS = -I$(CORE) $(addprefix -I,$(LIB_DIRS))
CFLAGS = $(CDEFS) -g -Os -w -fno-exceptions -ffunction-sections -fdata-sections $(CINCS)
CXXFLAGS = $(CFLAGS)
#ASFLAGS = -Wa,-adhlns=$(<:.S=.lst),-gstabs 
LDFLAGS = -lm

# Programming support using avrdude. Settings and variables.
AVRDUDE_PORT = $(PORT)
AVRDUDE_WRITE_FLASH = -U flash:w:build/$(TARGET).hex
AVRDUDE_FLAGS = -V -F  \
-p $(ADMCU) -P $(AVRDUDE_PORT) -c $(AVRDUDE_PROGRAMMER) \
-b $(UPLOAD_RATE)

# Program settings
CC = $(AVR_TOOLS_PATH)/avr-gcc
CXX = $(AVR_TOOLS_PATH)/avr-g++
OBJCOPY = $(AVR_TOOLS_PATH)/avr-objcopy
OBJDUMP = $(AVR_TOOLS_PATH)/avr-objdump
AR  = $(AVR_TOOLS_PATH)/avr-ar
SIZE = $(AVR_TOOLS_PATH)/avr-size
NM = $(AVR_TOOLS_PATH)/avr-nm
AVRDUDE = $(AVR_TOOLS_PATH)/avrdude
REMOVE = rm -f
MV = mv -f

# Define all object files.

OBJS = $(filter %.o,$(ALL_SRCS:.c=.o) $(ALL_SRCS:.cpp=.o))

# Combine all necessary flags and optional flags.
# Add target processor to flags.
ALL_CFLAGS = -mmcu=$(MCU) -I. $(CFLAGS)
ALL_CXXFLAGS = -mmcu=$(MCU) -I. $(CXXFLAGS)
ALL_ASFLAGS = -mmcu=$(MCU) -I. -x assembler-with-cpp $(ASFLAGS)


# Default target.
all: elf hex sizeafter

build:
	mkdir build

build/$(TARGET).cpp: build $(TARGET).pde
	echo '#include "WProgram.h"' > build/$(TARGET).cpp
	cat $(TARGET).pde >> build/$(TARGET).cpp
	cat $(CORE)/main.cpp >> build/$(TARGET).cpp

build_files: build/$(TARGET).cpp

elf: build/$(TARGET).elf
hex: build/$(TARGET).hex
eep: build/$(TARGET).eep
lss: build/$(TARGET).lss 
sym: build/$(TARGET).sym

# Program the device.  
upload: build/$(TARGET).hex
	$(AVRDUDE) $(AVRDUDE_FLAGS) $(AVRDUDE_WRITE_FLASH)


	# Display size of file.
HEXSIZE = $(SIZE) --target=$(FORMAT) build/$(TARGET).hex
ELFSIZE = $(SIZE)  build/$(TARGET).elf
sizebefore:
	@if [ -f build/$(TARGET).elf ]; then echo; echo $(MSG_SIZE_BEFORE); $(HEXSIZE); echo; fi

sizeafter:
	@if [ -f build/$(TARGET).elf ]; then echo; echo $(MSG_SIZE_AFTER); $(HEXSIZE); echo; fi


# Convert ELF to COFF for use in debugging / simulating in AVR Studio or VMLAB.
COFFCONVERT=$(OBJCOPY) --debugging \
--change-section-address .data-0x800000 \
--change-section-address .bss-0x800000 \
--change-section-address .noinit-0x800000 \
--change-section-address .eeprom-0x810000 


coff: build/$(TARGET).elf
	$(COFFCONVERT) -O coff-avr build/$(TARGET).elf $(TARGET).cof


extcoff: $(TARGET).elf
	$(COFFCONVERT) -O coff-ext-avr build/$(TARGET).elf $(TARGET).cof


.SUFFIXES: .elf .hex .eep .lss .sym

.elf.hex:
	$(OBJCOPY) -O $(FORMAT) -R .eeprom $< $@

.elf.eep:
	-$(OBJCOPY) -j .eeprom --set-section-flags=.eeprom="alloc,load" \
	--change-section-lma .eeprom=0 -O $(FORMAT) $< $@

# Create extended listing file from ELF output file.
.elf.lss:
	$(OBJDUMP) -h -S $< > $@

# Create a symbol table from ELF output file.
.elf.sym:
	$(NM) -n $< > $@

	# Link: create ELF output file from library.
build/$(TARGET).elf: $(OBJS) build/$(TARGET).o
	$(CC) -mmcu=$(MCU) -Os -Wl,--gc-sections -o $@ build/$(TARGET).o $(LDFLAGS) $(OBJS)


# Compile: create object files from C++ source files.
.cpp.o:
	$(CXX) -c $(ALL_CXXFLAGS) $< -o $@ 

# Compile: create object files from C source files.
.c.o:
	$(CC) -c $(ALL_CFLAGS) $< -o $@ 


# Compile: create assembler files from C source files.
.c.s:
	$(CC) -S $(ALL_CFLAGS) $< -o $@


# Assemble: create object files from assembler source files.
.S.o:
	$(CC) -c $(ALL_ASFLAGS) $< -o $@



# Target: clean project.
clean:
	rm -rf build
	rm -f $(OBJS)

depend:
	if grep '^# DO NOT DELETE' $(MAKEFILE) >/dev/null; \
	then \
		sed -e '/^# DO NOT DELETE/,$$d' $(MAKEFILE) > \
			$(MAKEFILE).$$$$ && \
		$(MV) $(MAKEFILE).$$$$ $(MAKEFILE); \
	fi
	echo '# DO NOT DELETE THIS LINE -- make depend depends on it.' \
		>> $(MAKEFILE); \
	$(CC) -M -mmcu=$(MCU) $(CDEFS) $(CINCS) $(SRC) $(ASRC) >> $(MAKEFILE)

# Debug messages
debug:
	# ARDUINO_HOME is $(ARDUINO_HOME)
	# AVR_TOOLS_PATH is $(AVR_TOOLS_PATH)
	# LIB_DIRS is $(LIB_DIRS)
	# LIB_SRCS is $(LIB_SRCS)
	# CORE_SRCS is $(CORE_SRCS)

.PHONY:	all elf hex eep lss sym program coff extcoff clean depend build_files sizebefore sizeafter debug
