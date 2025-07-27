# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This repository contains two primary components for the MiSTer FPGA platform:

### Main_MiSTer-master/
The main binary and framework for MiSTer. This is a C/C++ application that runs on the ARM core of the DE10-Nano FPGA board.

**Key Components:**
- `main.cpp` - Entry point, handles core affinity and initialization
- `menu.cpp/h` - Main menu system and UI logic
- `user_io.cpp/h` - User input/output handling
- `fpga_io.cpp/h` - FPGA communication interface
- `video.cpp/h` - Video processing and output
- `audio.cpp/h` - Audio processing and output
- `file_io.cpp/h` - File system operations
- `support/` - Core-specific implementations for various systems (Minimig, C64, N64, PSX, etc.)
- `lib/` - Third-party libraries (libco, miniz, lzma, zstd, libchdr, bluetooth)

### Template_MiSTer-master/
FPGA core template using SystemVerilog/Verilog for creating new MiSTer cores.

**Key Components:**
- `mycore.sv` - Main core wrapper and glue logic
- `sys/` - Framework files (DO NOT MODIFY - updated automatically)
- `rtl/` - Core-specific RTL implementation
- `files.qip` - Quartus IP file list (manually maintained)
- `mycore.qsf/.qpf` - Quartus project files

## Build Commands

### Main_MiSTer (ARM Binary)
```bash
# Build the main binary
make

# Clean build artifacts
make clean

# Debug build
make DEBUG=1

# Profiling build
make PROFILING=1

# Build and deploy to MiSTer (requires network setup)
./build.sh
```

### Template_MiSTer (FPGA Core)
- Use **Quartus v17.0.x** (specifically 17.0.2 recommended)
- Open `mycore.qpf` in Quartus Prime
- Compile using Quartus IDE
- Generated `.rbf` files go in `releases/` folder with format `<core_name>_YYYYMMDD.rbf`

## Development Environment

### ARM Cross-Compilation
- Uses `arm-none-linux-gnueabihf-gcc` toolchain (version 10.2.1)
- Parallel compilation enabled (`-j $(nproc)`)
- Required libraries: bluetooth, pthread, imlib2, freetype, libpng

### FPGA Development
- **Must use Quartus v17.0.x** - newer versions cause compatibility issues
- Framework located in `sys/` folder - never modify these files
- PLL required in `rtl/pll/` with `pll.v` and `pll.qip`
- Use `files.qip` for file management, not Quartus project settings

## Architecture Overview

### Main_MiSTer Architecture
- **Main Loop**: Event-driven architecture with scheduler
- **Core Communication**: Bidirectional communication between ARM and FPGA cores
- **Core Support**: Modular support system for different retro systems
- **File I/O**: Handles disk images, ROM files, and save states
- **Video Pipeline**: Supports multiple output formats (HDMI, VGA, composite)
- **Input System**: Unified input handling for keyboards, joysticks, and mice

### Framework Macros (Template_MiSTer)
Key preprocessor macros that affect framework behavior:
- `MISTER_DEBUG_NOHDMI` - Disable HDMI for faster compilation
- `MISTER_DUAL_SDRAM` - Enable dual SDRAM configuration
- `MISTER_FB` - Enable framebuffer support
- `MISTER_SMALL_VBUF` - Reduce video buffer size
- `MISTER_DOWNSCALE_NN` - Enable downscaling

## Core-Specific Notes

### Supported Systems
The framework supports numerous retro systems through modular implementations:
- **Minimig** (Amiga) - `support/minimig/`
- **Archie** (Acorn Archimedes) - `support/archie/`
- **C64** (Commodore 64) - `support/c64/`
- **PSX** (PlayStation) - `support/psx/`
- **N64** (Nintendo 64) - `support/n64/`
- **Saturn** (Sega Saturn) - `support/saturn/`
- **Arcade** cores - `support/arcade/`

### File Format Support
- **CHD** - Compressed hard disk images
- **ISO/CUE/BIN** - CD-ROM images
- **Various disk formats** - ADF, DSK, VHD, etc.
- **Compressed files** - ZIP, LZMA, ZSTD support

## Important Guidelines

### Code Standards
- C99 for C files, C++14 for C++ files
- ARM-specific optimizations enabled
- Thread affinity: main process pinned to core #1
- Error handling through return codes and logging

### FPGA Core Standards
- All cores must include unmodified `sys/` framework
- Use `files.qip` for file management
- Follow naming conventions: `<core_name>_YYYYMMDD.rbf`
- Maintain compatibility with Quartus 17.0.x

### Security Considerations
- No hardcoded credentials (uses configurable host file for deployment)
- File access restricted to appropriate directories
- Input validation for file operations