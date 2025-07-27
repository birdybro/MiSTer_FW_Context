# MiSTer Framework Comprehensive Documentation

## Table of Contents

1. [Overview](#overview)
2. [ARM Binary Architecture](#arm-binary-architecture)
3. [FPGA Framework Components](#fpga-framework-components)
4. [Core Communication Protocols](#core-communication-protocols)
5. [Video Subsystem Architecture](#video-subsystem-architecture)
6. [Audio Subsystem Architecture](#audio-subsystem-architecture)
7. [Input System](#input-system)
8. [File I/O and Storage Systems](#file-io-and-storage-systems)
9. [Core-Specific Implementations](#core-specific-implementations)
10. [Build System and Toolchain](#build-system-and-toolchain)
11. [Development Guidelines](#development-guidelines)

## Overview

The MiSTer framework is a sophisticated FPGA-based retro computing platform that combines ARM software running on the Cyclone V HPS (Hard Processor System) with custom FPGA cores implementing various retro computer and console systems. The framework provides a unified interface for hardware abstraction, user interaction, and system management.

### Key Components
- **Main_MiSTer**: ARM-based main application and framework
- **Template_MiSTer**: FPGA core template and system framework
- **Support Modules**: Core-specific implementations for various retro systems

## ARM Binary Architecture

### Core Application Structure

The main ARM application (`main.cpp`) implements a cooperative multitasking system with dedicated threads for different responsibilities:

```cpp
// CPU affinity: Pin main process to core #1
cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(1, &set);
sched_setaffinity(0, sizeof(set), &set);
```

### Key Modules

#### Menu System (`menu.cpp/h`)
- **Purpose**: Main user interface and file browser
- **Architecture**: State machine-driven hierarchical menu system
- **Key Functions**:
  - `HandleUI()` - Main UI event loop
  - `SelectFile()` - File browser implementation
  - `ProgressMessage()` - User feedback system
  - `InfoMessage()` - Status notifications

#### User I/O Interface (`user_io.cpp/h`)
The user I/O system provides the primary communication layer between ARM and FPGA:

**Command Codes**:
```cpp
#define UIO_STATUS      0x00  // Status register access
#define UIO_JOYSTICK0   0x02  // Joystick data transmission
#define UIO_MOUSE       0x04  // Mouse data transmission
#define UIO_KEYBOARD    0x05  // Keyboard data transmission
#define UIO_SECTOR_RD   0x17  // SD card sector read
#define UIO_SECTOR_WR   0x18  // SD card sector write
#define UIO_SET_VIDEO   0x20  // Video configuration
#define UIO_RTC         0x22  // Real-time clock data
```

**Core Detection System**:
```cpp
unsigned char user_io_core_type();  // Returns core type identifier
char* user_io_get_core_name();      // Returns human-readable core name
```

#### FPGA I/O Interface (`fpga_io.cpp/h`)
Low-level FPGA communication through SPI and memory-mapped registers:

**SPI Communication**:
```cpp
uint16_t fpga_spi(uint16_t word);                    // Single word transfer
void fpga_spi_fast_block_write(const uint16_t *buf, uint32_t length);  // Block transfer
void fpga_spi_fast_block_read(uint16_t *buf, uint32_t length);         // Block read
```

**Core Management**:
```cpp
void fpga_core_reset(int reset);                     // Reset FPGA core
void fpga_core_write(uint32_t offset, uint32_t value); // Write to core registers
uint32_t fpga_core_read(uint32_t offset);            // Read from core registers
int fpga_load_rbf(const char *name);                 // Load FPGA core from RBF file
```

### Scheduler System (`scheduler.cpp/h`)

The framework uses cooperative multitasking based on libco:

```cpp
// Thread structure
typedef struct {
    cothread_t thread;
    void (*func)(void);
    const char *name;
} thread_t;

// Main threads
static thread_t threads[] = {
    { 0, poll_thread, "poll" },    // Hardware polling
    { 0, ui_thread, "ui" }         // User interface
};
```

**Thread Responsibilities**:
- **Poll Thread**: Input processing, FPGA communication, hardware monitoring
- **UI Thread**: Menu handling, OSD updates, user interactions

## FPGA Framework Components

### System Top Module (`sys_top.v`)

The system top module provides hardware abstraction for the DE10-Nano board:

**I/O Interfaces**:
- **Clock Inputs**: 3x 50MHz clocks (FPGA_CLK1_50, FPGA_CLK2_50, FPGA_CLK3_50)
- **HDMI Output**: Full digital video/audio interface
- **SDRAM Interface**: 16-bit SDRAM controller
- **VGA Output**: Analog video output (when not using dual SDRAM)
- **Audio Outputs**: Analog stereo + S/PDIF
- **User Interface**: LEDs, buttons, and OSD control

**Conditional Compilation**:
```systemverilog
`ifdef MISTER_DUAL_SDRAM
    // Dual SDRAM configuration
    output [12:0] SDRAM2_A,
    inout  [15:0] SDRAM2_DQ,
    // ... secondary SDRAM signals
`else
    // VGA and audio outputs
    output  [5:0] VGA_R,
    output  [5:0] VGA_G,
    output  [5:0] VGA_B,
    // ... analog outputs
`endif
```

### HPS I/O Module (`hps_io.sv`)

The HPS I/O module handles communication between ARM and FPGA:

**Key Parameters**:
- `CONF_STR`: Configuration string defining OSD menu structure
- `WIDE`: 16-bit file I/O mode (0=8-bit, 1=16-bit)
- `VDNUM`: Number of virtual drives (1-10)
- `BLKSZ`: Block size for file operations (0=128B ... 7=16KB)

**Interface Signals**:
```systemverilog
// Joystick interfaces (6 controllers, 32 buttons each)
output reg [31:0] joystick_0, joystick_1, ..., joystick_5,

// Analog stick support (left/right per controller)
output reg [15:0] joystick_l_analog_0, ..., joystick_l_analog_5,
output reg [15:0] joystick_r_analog_0, ..., joystick_r_analog_5,

// PS2 keyboard/mouse emulation
output ps2_kbd_clk_out, ps2_kbd_data_out,
input  ps2_kbd_clk_in,  ps2_kbd_data_in,

// Alternative PS2 interface with scan codes
output reg [10:0] ps2_key,  // [10]=toggle, [9]=pressed, [8]=extended
output reg [24:0] ps2_mouse, // [24]=toggle, [15:8]=Y, [7:0]=X
```

### Video Processing Pipeline

#### Video Mixer (`video_mixer.sv`)
Multi-stage video processing with support for:
- **Gamma Correction**: LUT-based gamma correction
- **Scandoubler**: Line doubling for VGA output
- **HQ2X Scaling**: High-quality 2x pixel art scaling
- **Freeze Control**: HDMI freeze (last frame) or VGA freeze (black)

**Processing Chain**:
```
Input RGB → Freeze Control → Gamma Correction → Scandoubler → HQ2X → Output RGB
```

#### Video Cleaner (`video_cleaner.sv`)
Sync signal conditioning and timing alignment:
- Auto-detection of sync polarity
- Interlace support with field alignment
- DE (Data Enable) signal generation
- Blank signal conditioning

#### Video Freak (`video_freak.sv`)
Advanced scaling and cropping engine:
- Dynamic crop calculation
- Integer scaling modes (V-integer, HV-Integer variants)
- Aspect ratio preservation
- HDMI resolution-aware scaling

### Audio Processing System

#### Audio Output (`audio_out.v`)
Comprehensive audio processing chain:

**Sample Rate Support**:
- 48 kHz (standard)
- 96 kHz (high-quality mode)

**Processing Stages**:
1. **Input Stabilization**: Double-register synchronization
2. **IIR Filtering**: Programmable coefficients for audio enhancement
3. **DC Blocking**: High-pass filter to remove DC offset
4. **Audio Mixing**: Core audio + ALSA system audio
5. **Attenuation Control**: Digital volume control

**Output Formats**:
- I2S digital audio
- S/PDIF digital audio
- Sigma-Delta analog audio

#### I2S Interface (`i2s.v`)
Standard I2S serial audio protocol implementation:
- Bit clock generation from system clock
- Frame sync (LR clock) for channel selection
- MSB-first data serialization

### Memory Architecture

#### DDR Service Controller (`ddr_svc.sv`)
16-bit DDR memory arbitration for dual-channel access:
- **Interface**: Avalon-MM burst-capable
- **Arbitration**: Round-robin between channels
- **Burst Support**: 1-255 word bursts
- **Address Space**: 29-bit (supporting up to 1GB)
- **Data Width**: 64-bit data bus

#### System Memory (`sysmem.sv`)
HPS-FPGA bridge interface management:
- **RAM Interfaces**: 2x 64-bit standard RAM interfaces
- **Video Buffer**: 1x 128-bit video buffer interface
- **Reset Management**: Multi-domain reset synchronization
- **Safe Termination**: Prevents bus lockup during reset

#### SD Card Controller (`sd_card.sv`)
Full SPI-based SD card implementation:
- **Card Support**: SDHC and SDSC
- **Operations**: Multi-block read/write
- **Emulation**: CSD/CID register emulation
- **Performance**: Prefetch mechanism for improved throughput
- **Clock Domains**: System/SPI clock domain crossing with dual-port memory

## Core Communication Protocols

### Command Structure

Communication between ARM and FPGA follows a command-response protocol:

**Command Format**:
```
Byte 0: Command Code (UIO_*)
Byte 1: Data Length
Bytes 2-N: Command Data
```

### Key Communication Patterns

#### Status Register Access
```cpp
// Read core status
uint32_t status = user_io_status_get("option_name");

// Write core configuration
user_io_status_set("option_name", value);
```

#### File Transfer Protocol
```cpp
// Initiate file transfer to FPGA
user_io_file_tx(filename, index, opensave, mute, composite, load_addr);

// Set download mode
user_io_set_download(enable, address);

// Transfer data blocks
user_io_file_tx_data(data_buffer, length);
```

#### Real-Time Data Streams
```cpp
// Joystick data transmission
user_io_digital_joystick(controller_id, button_state, analog_data);

// Mouse movement
user_io_mouse(buttons, delta_x, delta_y, wheel);

// Keyboard scan codes
user_io_kbd(scancode, press_release);
```

## Video Subsystem Architecture

### Video Information Structure
```cpp
struct VideoInfo {
    uint32_t width, height;     // Resolution
    uint32_t htime, vtime;      // Timing parameters
    uint32_t arx, ary;          // Aspect ratio
    uint32_t fb_en;             // Framebuffer enable
    uint32_t fb_fmt;            // Framebuffer format
    bool interlaced;            // Interlace mode
    bool rotated;               // Rotation flag
};
```

### Scaling and Filtering

#### Scaler Filter Types
```cpp
#define VFILTER_HORZ  0  // Horizontal filtering
#define VFILTER_VERT  1  // Vertical filtering
#define VFILTER_SCAN  2  // Scanline generation
#define VFILTER_ILACE 3  // Interlace processing
```

#### Filter Configuration
```cpp
int video_get_scaler_flt(int type);                    // Get current filter
void video_set_scaler_flt(int type, int filter_num);   // Set predefined filter
void video_set_scaler_coeff(int type, const char *name); // Load custom coefficients
```

### Gamma Correction System
```cpp
int video_get_gamma_en();                    // Check gamma enable status
void video_set_gamma_en(int enable);         // Enable/disable gamma
char* video_get_gamma_curve(int only_name);  // Get current gamma curve
void video_set_gamma_curve(const char *name); // Set gamma curve
```

### Shadow Mask Support
```cpp
int video_get_shadow_mask_mode();             // Get shadow mask mode
void video_set_shadow_mask_mode(int mode);    // Set shadow mask mode
char* video_get_shadow_mask(int only_name);   // Get current mask
void video_set_shadow_mask(const char *name); // Set shadow mask
```

## Audio Subsystem Architecture

### Volume Control System
```cpp
void set_volume(int cmd);        // Set master volume
int get_volume();                // Get master volume
int get_core_volume();           // Get core-specific volume
void set_core_volume(int cmd);   // Set core-specific volume
```

### Audio Filtering
```cpp
int audio_filter_en();                      // Check filter enable status
char* audio_get_filter(int only_name);      // Get current filter
void audio_set_filter(const char *name);    // Set audio filter
void audio_set_filter_en(int enable);       // Enable/disable filtering
```

### Audio Processing Chain
1. **Core Audio**: Generated by FPGA core
2. **IIR Filtering**: Configurable frequency response
3. **Volume Control**: Digital attenuation
4. **Mixing**: Core audio + system audio
5. **Output**: I2S, S/PDIF, or analog

## Input System

### Device Architecture

#### Input Device Structure
The input system supports up to 30 simultaneous devices with real-time hotplug detection:

```cpp
typedef struct {
    int fd;                    // File descriptor
    char device_name[256];     // Device name
    char device_type[64];      // Device type
    uint32_t capabilities;     // Device capabilities
    int axis_map[ABS_CNT];    // Axis mapping
    int button_map[KEY_CNT];  // Button mapping
} input_device_t;
```

#### Supported Device Types
- **Keyboards**: Full scancode mapping with multiple layouts
- **Mice**: 3-button + wheel support
- **Gamepads**: 32 buttons + analog sticks per controller
- **Light Guns**: Calibrated absolute positioning
- **Paddles**: Analog paddle controllers (Atari-style)
- **Spinners**: Rotary encoders for driving games

### Input Translation System

#### Keyboard Mapping
Multiple scancode tables for different systems:
```cpp
// PS2 scancode translation
void kbd_translate_ps2(uint16_t code, int press);

// Amiga scancode translation  
void kbd_translate_amiga(uint16_t code, int press);

// System-specific translations
void kbd_translate_archie(uint16_t code, int press);
```

#### Controller Mapping
```cpp
// Digital joystick state (32 buttons)
void user_io_digital_joystick(unsigned char num, uint64_t state, int analog);

// Analog stick positions (-127 to +127)
void user_io_l_analog_joystick(unsigned char num, char x, char y);
void user_io_r_analog_joystick(unsigned char num, char x, char y);
```

### Button Mapping System

#### Virtual Gamepad Layout
```cpp
// Standard button mapping
#define JOY_A      JOY_BTN1    // Primary action
#define JOY_B      JOY_BTN2    // Secondary action
#define JOY_SELECT JOY_BTN3    // Select/back
#define JOY_START  JOY_BTN4    // Start/menu
#define JOY_X      0x100       // Extended buttons
#define JOY_Y      0x200
#define JOY_L      0x400       // Shoulder buttons
#define JOY_R      0x800
#define JOY_L2     0x1000      // Triggers
#define JOY_R2     0x2000
```

## File I/O and Storage Systems

### Core File Abstraction

#### fileTYPE Structure
```cpp
typedef struct {
    FILE *filp;               // Standard file pointer
    int mode;                 // Access mode (read/write)
    int type;                 // File type identifier
    fileZipArchive *zip;      // ZIP archive context
    __off64_t size;           // File size in bytes
    __off64_t offset;         // Current file position
    char path[1024];          // Full file path
    char name[261];           // Filename only
} fileTYPE;
```

#### File Operations
```cpp
int FileOpen(fileTYPE *file, const char *name, int mode);  // Open file
int FileClose(fileTYPE *file);                             // Close file
int FileRead(fileTYPE *file, void *buffer, int length);    // Read data
int FileWrite(fileTYPE *file, void *buffer, int length);   // Write data
int FileSeek(fileTYPE *file, __off64_t offset, int origin); // Seek position
```

### ZIP Archive Support

#### Archive Management
```cpp
typedef struct {
    unzFile file;             // ZIP file handle
    char path[1024];          // Archive path
    int index;                // Current file index
    unz_file_info64 info;     // File information
} fileZipArchive;
```

#### ZIP Operations
- Cached ZIP operations to prevent multiple opens
- Integrated directory scanning
- Transparent file access (treats ZIP contents as regular files)
- Automatic decompression for supported formats

### Storage System Architecture

#### Multi-Storage Support
- **Primary Storage**: SD card or eMMC
- **USB Storage**: External USB drives
- **Network Storage**: CIFS/SMB shares
- **Storage Migration**: Automatic file movement between storage types

#### Directory Structure
```
/media/fat/
├── saves/          # Save states and save files
├── config/         # Core configurations
├── games/          # ROM/game files by system
│   ├── Amiga/
│   ├── C64/
│   ├── Arcade/
│   └── ...
├── filters/        # Video/audio filter coefficients
└── docs/          # Documentation files
```

## Core-Specific Implementations

### Minimig (Amiga) Support

#### Memory Configuration
```cpp
typedef struct {
    uint32_t chip_mem;        // Chip memory size (512K-2MB)
    uint32_t slow_mem;        // Slow memory size (0-1.8MB)
    uint32_t fast_mem;        // Fast memory size (0-8MB)
} minimig_memory_config_t;
```

#### Chipset Configuration
```cpp
typedef struct {
    int chipset;              // OCS/ECS/AGA
    int cpu_type;             // 68000/68010/68020
    int kickstart;            // Kickstart ROM version
    int ntsc_pal;             // Video standard
} minimig_chipset_config_t;
```

#### Floppy Drive Support
- ADF (Amiga Disk Format) support
- Extended ADF with track information
- Write protection handling
- Drive LED emulation

### Arcade Support

#### MRA (MAME ROM Archive) System
```cpp
typedef struct {
    char name[256];           // ROM set name
    char md5[33];             // MD5 checksum
    uint32_t load_addr;       // Load address
    uint32_t length;          // ROM length
    int interleave;           // Interleave pattern
} arcade_rom_t;
```

#### NVRAM Management
- Automatic NVRAM save/load
- High score preservation
- Game-specific NVRAM handling
- Backup and restore functionality

### CD-ROM Support (Multiple Cores)

#### CHD (Compressed Hunks of Data) Support
```cpp
typedef struct {
    chd_file *file;           // CHD file handle
    uint32_t total_hunks;     // Number of hunks
    uint32_t hunk_size;       // Hunk size in bytes
    uint64_t logical_bytes;   // Logical size
} chd_context_t;
```

#### CD-ROM Image Formats
- **ISO**: Standard ISO 9660 filesystem
- **CUE/BIN**: Cue sheet + binary image
- **CHD**: Compressed format with metadata
- **Track Support**: Audio and data tracks

## Build System and Toolchain

### ARM Toolchain

#### GCC Cross-Compiler Setup
```bash
# Default toolchain configuration
MISTER_GCC_VER=10.2-2020.11
MISTER_GCC_HOST_ARCH=x86_64
GCC_PACKAGE_NAME=gcc-arm-$MISTER_GCC_VER-$MISTER_GCC_HOST_ARCH-arm-none-linux-gnueabihf
```

#### Makefile Configuration
```makefile
# Compiler settings
CC = arm-none-linux-gnueabihf-gcc
CFLAGS = -Wall -Wextra -O3 -std=gnu99
LFLAGS = -lc -lstdc++ -lm -lrt -lbluetooth -lpthread

# Library includes
INCLUDE += -I./lib/libco
INCLUDE += -I./lib/miniz
INCLUDE += -I./lib/lzma
INCLUDE += -I./lib/zstd/lib
```

### FPGA Build System

#### Quartus Integration
**Required Version**: Quartus Prime 17.0.x (specifically 17.0.2)

#### TCL Build Scripts

**Build ID Generation** (`build_id.tcl`):
```tcl
proc generateBuildID_Verilog {} {
    set buildDate "`define BUILD_DATE \"[clock format [clock seconds] -format %y%m%d]\""
    # Write to build_id.v for inclusion in design
}
```

**JTAG Configuration** (`build_id.tcl`):
```tcl
proc generateCDF {revision device outpath} {
    # Generate JTAG Chain Description File
    # Enables programming via USB Blaster
}
```

### Project File Management

#### Files.qip Structure
Manual file management for Quartus projects:
```qip
set_global_assignment -name SYSTEMVERILOG_FILE rtl/mycore.sv
set_global_assignment -name VERILOG_FILE rtl/cpu/cpu_core.v
set_global_assignment -name VERILOG_FILE rtl/video/video_gen.v
```

#### Device Configuration (`sys.tcl`)
```tcl
set_global_assignment -name FAMILY "Cyclone V"
set_global_assignment -name DEVICE 5CSEBA6U23I7
set_global_assignment -name DEVICE_FILTER_PACKAGE UFBGA
set_global_assignment -name DEVICE_FILTER_PIN_COUNT 672
```

### Clean-up Scripts

#### ARM Build Cleanup (`clean.sh`)
```bash
#!/bin/bash
make clean
exit 0
```

#### FPGA Build Cleanup (`clean.bat`)
```batch
del /s *.bak *.orig *.rej *~
rmdir /s /q db incremental_db output_files simulation
```

## Development Guidelines

### Code Standards

#### C/C++ Guidelines
- **C Standard**: C99 for C files
- **C++ Standard**: C++14 for C++ files
- **Naming**: Snake_case for functions, camelCase for variables
- **Error Handling**: Return codes and comprehensive logging
- **Memory Management**: RAII patterns, careful pointer management

#### SystemVerilog/Verilog Guidelines
- **Naming**: lowercase with underscores
- **Clock Domains**: Clear clock domain separation
- **Reset Strategy**: Synchronous reset preferred
- **Parameterization**: Extensive use of parameters for reusability

### Framework Macros

#### Conditional Compilation
```systemverilog
`ifdef MISTER_DEBUG_NOHDMI      // Disable HDMI for faster compilation
`ifdef MISTER_DUAL_SDRAM        // Enable dual SDRAM configuration
`ifdef MISTER_FB                // Enable framebuffer support
`ifdef MISTER_SMALL_VBUF        // Reduce video buffer size
`ifdef MISTER_DOWNSCALE_NN      // Enable downscaling support
`ifdef MISTER_DISABLE_ADAPTIVE  // Disable adaptive scanlines
`ifdef MISTER_FB_PALETTE        // Framebuffer palette support
```

### Core Development Process

#### 1. Template Setup
- Copy Template_MiSTer as starting point
- Rename files according to core name
- Update project settings in QSF/QPF files

#### 2. RTL Implementation
- Implement core logic in `rtl/` directory
- Ensure PLL configuration in `rtl/pll/`
- Update `files.qip` with all source files

#### 3. Framework Integration
- Adapt signals in main core SV file
- Configure HPS_IO parameters
- Implement configuration string (CONF_STR)

#### 4. Testing and Validation
- Use Quartus compilation for syntax checking
- Implement test benches for critical modules
- Validate on actual hardware

#### 5. Release Management
- Generate RBF files with date stamps
- Place in `releases/` directory
- Update documentation

### Performance Optimization

#### ARM Application
- **CPU Affinity**: Pin main thread to core #1
- **Cooperative Multitasking**: Minimize context switches
- **Memory Management**: Pool allocation for frequent operations
- **I/O Optimization**: Buffered file operations

#### FPGA Design
- **Clock Domain Crossing**: Proper CDC techniques
- **Memory Interface**: Efficient SDRAM/DDR usage
- **Resource Utilization**: Balance between LEs, RAM, and DSP blocks
- **Timing Closure**: Meet timing requirements across all clock domains

### Security Considerations

#### File Access Control
- Validate all file paths before access
- Prevent directory traversal attacks
- Implement proper permission checking

#### Network Security
- Secure configuration for network storage
- No hardcoded credentials
- Configurable network parameters

#### Input Validation
- Sanitize all user inputs
- Validate file formats before processing
- Bounds checking for all array accesses

This comprehensive documentation provides developers with the detailed technical information needed to understand, maintain, and extend the MiSTer framework effectively.