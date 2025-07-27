# MiSTer Configuration String (CONF_STR) System - Complete Technical Guide

## Table of Contents

1. [Overview](#overview)
2. [FPGA Side Implementation](#fpga-side-implementation)
3. [ARM Side Implementation](#arm-side-implementation)
4. [Configuration String Syntax](#configuration-string-syntax)
5. [Status Register System](#status-register-system)
6. [Menu Generation and OSD](#menu-generation-and-osd)
7. [File Dialog Integration](#file-dialog-integration)
8. [Conditional Display System](#conditional-display-system)
9. [Data Flow and Communication](#data-flow-and-communication)
10. [Advanced Features](#advanced-features)
11. [Best Practices and Examples](#best-practices-and-examples)

## Overview

The MiSTer Configuration String (CONF_STR) system is a sophisticated bidirectional communication framework that bridges the FPGA core and ARM-side Linux system. It provides a standardized method for defining menu structures, status registers, file dialogs, and on-screen display (OSD) elements through a semicolon-delimited string format.

### Key Features
- **Declarative Menu Definition**: Define complex menu hierarchies in a simple string format
- **Bidirectional Communication**: FPGA → ARM configuration definition, ARM → FPGA status updates
- **Dynamic Menu Control**: Runtime show/hide/disable menu items based on core state
- **File System Integration**: Built-in file browser and storage management
- **Multi-Page Support**: Organize complex configurations across multiple pages
- **Version Control**: Configuration compatibility tracking

## FPGA Side Implementation

### Configuration String Definition

In the FPGA template (`Template_MiSTer-master/mycore.sv`), the configuration string is defined as a SystemVerilog parameter:

```systemverilog
localparam CONF_STR = {
    "MyCore;;",                                              // Core name
    "-;",                                                    // Separator
    "O[122:121],Aspect ratio,Original,Full Screen,[ARC1],[ARC2];", // Multi-bit option
    "O[2],TV Mode,NTSC,PAL;",                               // Single-bit option
    "O[4:3],Noise,White,Red,Green,Blue;",                   // Multi-value option
    "-;",                                                    // Separator
    "P1,Test Page 1;",                                       // Page definition
    "P1O[5],Option 1-1,Off,On;",                           // Page-specific option
    "d0P1F1,BIN;",                                          // Conditional file dialog
    "-;",
    "T[0],Reset;",                                          // Momentary trigger
    "R[0],Reset and close OSD;",                           // Reset with menu close
    "v,0;",                                                 // Config version
    "V,v",`BUILD_DATE                                       // Version display
};
```

### HPS I/O Integration

The configuration string is passed to the `hps_io` module, which handles communication with the ARM side:

```systemverilog
module hps_io #(
    parameter CONF_STR,           // Configuration string
    CONF_STR_BRAM=0,             // Store in BRAM (0=registers, 1=BRAM)
    PS2DIV=0,                    // PS2 clock divider
    WIDE=0,                      // File I/O width (0=8-bit, 1=16-bit)
    VDNUM=1,                     // Number of virtual drives
    BLKSZ=2,                     // Block size (0=128B, 1=256B, 2=512B, etc.)
    PS2WE=0,                     // PS2 write enable
    STRLEN=$size(CONF_STR)>>3    // String length in bytes
) (
    input             clk_sys,
    inout      [48:0] HPS_BUS,
    
    output reg [127:0] status,       // Status register output
    input      [127:0] status_in,    // Status register input
    output reg [127:0] status_set,   // Status bits to set
    output reg [127:0] status_menumask, // Menu visibility mask
    
    // ... other ports
);
```

### Status Register Connection

The core connects its configuration to the status register:

```systemverilog
wire [127:0] status;
wire forced_scandoubler;
wire [1:0] buttons;

hps_io #(.CONF_STR(CONF_STR)) hps_io
(
    .clk_sys(clk_sys),
    .HPS_BUS(HPS_BUS),
    
    .status(status),
    .status_menumask({status[5]}),  // Dynamic menu control
    .forced_scandoubler(forced_scandoubler),
    .buttons(buttons),
    
    // ... other connections
);

// Use status bits in core logic
wire [1:0] aspect_ratio = status[122:121];
wire tv_mode = status[2];
wire [1:0] noise_color = status[4:3];
wire reset = status[0] | buttons[1];
```

## ARM Side Implementation

### Configuration String Reading

The ARM side reads the configuration string through SPI communication (`Main_MiSTer-master/user_io.cpp`):

```cpp
static char cfgstr[1024 * 10] = {};

void user_io_read_confstr()
{
    spi_uio_cmd_cont(UIO_GET_STRING);
    
    uint32_t j = 0;
    while (j < sizeof(cfgstr) - 1)
    {
        char i = spi_in();
        if (!i) break;
        cfgstr[j++] = i;
    }
    
    cfgstr[j++] = 0;
    DisableIO();
}
```

### String Tokenization and Access

Individual configuration items are extracted using semicolon separation:

```cpp
char *user_io_get_confstr(int index)
{
    int lidx = 0;
    static char buffer[(1024*2) + 1];
    
    char *start = cfgstr;
    while (lidx < index)
    {
        start = strchr(start, ';');
        if (!start) return NULL;
        start++;
        lidx++;
    }
    
    char *end = strchr(start, ';');
    int len = end ? end - start : strlen(start);
    if (!len) return NULL;
    
    if ((uint32_t)len > sizeof(buffer) - 1) len = sizeof(buffer) - 1;
    memcpy(buffer, start, len);
    buffer[len] = 0;
    return buffer;
}
```

### Substring Extraction

The `substrcpy` function extracts specific parts of configuration entries:

```cpp
int substrcpy(char *d, const char *s, char idx)
{
    int pos = 0;
    int p = 0;
    int i = 0;
    
    while (s[p])
    {
        if (s[p] == ',')
        {
            if (i == idx)
            {
                d[pos] = 0;
                return pos;
            }
            i++;
            pos = 0;
        }
        else if (i == idx)
        {
            d[pos++] = s[p];
        }
        p++;
    }
    
    if (i == idx)
    {
        d[pos] = 0;
        return pos;
    }
    
    d[0] = 0;
    return 0;
}
```

## Configuration String Syntax

### Basic Structure

Each configuration entry is terminated by a semicolon (`;`) and follows this general format:
```
[Modifiers][Command][Parameters],Title,Option1,Option2,...;
```

### Core Information
```systemverilog
"MyCore;;",          // Core name (double semicolon)
```

### Menu Separators
```systemverilog
"-;",                // Horizontal separator line
```

### Comments and Labels
```systemverilog
"-, -= Section Title =-;",  // Section header with decorative formatting
```

## Command Reference

### Option Commands (O/o)

#### Single-bit Options
```systemverilog
"O[2],TV Mode,NTSC,PAL;"
```
- **Bit Address**: `[2]` = status bit 2
- **Title**: "TV Mode"
- **Options**: "NTSC" (value 0), "PAL" (value 1)

#### Multi-bit Options
```systemverilog
"O[4:3],Noise,White,Red,Green,Blue;"
```
- **Bit Range**: `[4:3]` = status bits 3-4 (2 bits = 4 values)
- **Values**: White (00), Red (01), Green (10), Blue (11)

#### Extended Status Options
```systemverilog
"o[5],Extended Option,Off,On;"
```
- Uses extended status register (bits 32-159)

### Toggle Commands (T/R)

#### Momentary Toggle
```systemverilog
"T[0],Reset;"
```
- Creates momentary pulse on status bit 0
- Returns to 0 automatically after trigger

#### Reset with Menu Close
```systemverilog
"R[0],Reset and close OSD;"
```
- Triggers reset and automatically closes menu

#### Extended Status Toggles
```systemverilog
"t[1],Extended Toggle;"
"r[1],Extended Reset;"
```
- Use extended status register space

### File Dialog Commands (F/S)

#### File Browser
```systemverilog
"F1,BIN;"              // File index 1, BIN extension
"F2,ROM,ROMX;"          // Multiple extensions
```

#### Storage Mount
```systemverilog
"S0,DSK;"               // Storage index 0, DSK files
"S1,HDF,VHD;"           // Hard disk images
```

### Page System (P)

#### Page Definition
```systemverilog
"P1,Video Settings;"    // Define page 1 with title
"P2,Audio Settings;"    // Define page 2
```

#### Page-specific Items
```systemverilog
"P1O[10],Scanlines,Off,25%,50%,75%;"   // Option on page 1
"P2O[15],Audio Filter,Off,On;"          // Option on page 2
```

### Conditional Display (H/D)

#### Hide Commands
```systemverilog
"H5O[20],Hidden Option,Off,On;"    // Hide when status[5] = 1
"h5O[21],Visible Option,Off,On;"   // Hide when status[5] = 0
```

#### Disable Commands
```systemverilog
"D3F1,ROM;"                        // Disable when status[3] = 1
"d3S0,DSK;"                        // Disable when status[3] = 0
```

#### Multiple Conditions
```systemverilog
"H5D3P1O[25],Complex Option,Off,On;"  // Hide if bit 5 set, disable if bit 3 set
```

### Version Commands (V/v)

#### Configuration Version
```systemverilog
"v,2;"                 // Configuration version 2 (0-99)
```
- Used for configuration compatibility
- Resets all options to defaults when version changes

#### Version Display
```systemverilog
"V,v20241201;"         // Display version string
"V,v",`BUILD_DATE      // Display build date
```

## Status Register System

### Bit Addressing Formats

#### Legacy Single Character (0-9, A-V)
```systemverilog
"O1,Option,Off,On;"     // Status bit 1
"OA,Option,Off,On;"     // Status bit 10 (A = 10)
"OV,Option,Off,On;"     // Status bit 31 (V = 31)
```

#### Bracket Notation (Recommended)
```systemverilog
"O[0],Option,Off,On;"       // Single bit 0
"O[15:8],Option,Val1,Val2;" // Bit range 8-15 (8 bits = 256 values)
"O[127],Option,Off,On;"     // High bit address
```

#### Extended Status Register
```systemverilog
"o[0],Extended,Off,On;"     // Extended status bit 32
"o[95],High Extended,Off,On;" // Extended status bit 127 (32+95)
```

### Status Register Access Functions

#### Reading Status Values
```cpp
uint32_t user_io_status_get(const char *opt, int ex = 0)
{
    int start, end;
    int size = user_io_status_bits(opt, &start, &end, ex);
    if (!size) return 0;
    
    // Extract bits from status register
    uint32_t x = (cur_status[end / 8] << 8) | cur_status[start / 8];
    x >>= start % 8;
    return x & ~(0xffffffff << size);
}
```

#### Writing Status Values
```cpp
void user_io_status_set(const char *opt, uint32_t value, int ex = 0)
{
    int start, end;
    int size = user_io_status_bits(opt, &start, &end, ex);
    if (!size) return;
    
    uint32_t mask = ~(0xffffffff << size);
    value &= mask;
    
    // Update status register
    for (int i = 0; i < size; i++)
    {
        int bit = start + i;
        if (value & (1 << i))
            cur_status[bit / 8] |= (1 << (bit % 8));
        else
            cur_status[bit / 8] &= ~(1 << (bit % 8));
    }
    
    // Send to FPGA
    user_io_send_status();
}
```

### Bit Parsing Implementation

```cpp
int user_io_status_bits(const char *opt, int *s, int *e, int ex, int single)
{
    uint32_t start = 0, end = 0;
    
    if (opt[0] == '[')
    {
        // Bracket notation: [start:end] or [bit]
        if (!single && sscanf(opt, "[%u:%u]", &end, &start) == 2)
        {
            if (start > 127 || end > 127 || end <= start) return 0;
        }
        else if (sscanf(opt, "[%u]", &start) == 1)
        {
            if (start > 127) return 0;
            end = start;
        }
        else return 0;
    }
    else
    {
        // Legacy notation
        if ((opt[0] >= '0') && (opt[0] <= '9'))
            start = opt[0] - '0';
        else if ((opt[0] >= 'A') && (opt[0] <= 'V'))
            start = opt[0] - 'A' + 10;
        else return 0;
        
        // Two-character range
        if (opt[1] && (opt[1] >= '0') && (opt[1] <= '9'))
        {
            end = opt[1] - '0';
            if (end <= start) return 0;
        }
        else
        {
            end = start;
        }
    }
    
    // Apply extended status offset
    if (ex)
    {
        start += 32;
        end += 32;
    }
    
    if (s) *s = start;
    if (e) *e = end;
    
    return 1 + end - start;
}
```

## Menu Generation and OSD

### Menu Processing Loop

The menu system processes the configuration string to generate the OSD (`Main_MiSTer-master/menu.cpp`):

```cpp
case MENU_GENERIC_MAIN1:
{
    hdmask = spi_uio_cmd16(UIO_GET_OSDMASK, 0);
    user_io_read_confstr();
    
    int i = 0;
    uint32_t entry = 0;
    char *p;
    
    while ((p = user_io_get_confstr(i++)))
    {
        int h = 0, d = 0;
        
        // Process hide/disable flags
        while ((p[0] == 'H' || p[0] == 'D' || p[0] == 'h' || p[0] == 'd') && strlen(p) > 2)
        {
            int flg = (hdmask & (1 << user_io_hd_mask(p + 1))) ? 1 : 0;
            if (p[0] == 'H') h |= flg;
            if (p[0] == 'h') h |= (flg ^ 1);
            if (p[0] == 'D') d |= flg;
            if (p[0] == 'd') d |= (flg ^ 1);
            p += 2;
        }
        
        if (h) continue; // Skip hidden items
        
        // Process menu item type
        if ((p[0] == 'O') || (p[0] == 'o'))
        {
            // Option menu item
            uint32_t x = user_io_status_get(p + 1, p[0] == 'o');
            substrcpy(s, p, 2 + x);
            MenuWrite(entry, s, menusub == entry, d);
        }
        else if ((p[0] == 'T') || (p[0] == 'R'))
        {
            // Toggle/Reset menu item
            substrcpy(s + 1, p, 1);
            s[0] = ' ';
            MenuWrite(entry, s, menusub == entry, d);
        }
        else if ((p[0] == 'F') || (p[0] == 'S'))
        {
            // File dialog menu item
            substrcpy(s + 1, p, 1);
            s[0] = get_image_name(user_io_ext_idx(p + 1, fs_pFileExt)) ? 0x16 : ' ';
            MenuWrite(entry, s, menusub == entry, d);
        }
        else if (p[0] == '-')
        {
            // Separator
            MenuWrite(entry, "", 0, 0);
        }
        
        entry++;
    }
}
```

### Option Value Display

For option items, the current value is displayed by extracting the corresponding choice text:

```cpp
if ((p[0] == 'O') || (p[0] == 'o'))
{
    uint32_t x = user_io_status_get(p + 1, p[0] == 'o');
    
    // Extract title
    if (!substrcpy(s, p, 1))
    {
        strcpy(s, "OPTION");
    }
    
    // Extract current option text
    if (substrcpy(opt, p, 2 + x))
    {
        sprintf(s + strlen(s), ": %s", opt);
    }
    
    MenuWrite(entry, s, menusub == entry, d);
}
```

### Menu Navigation

Menu navigation is handled through the status register and button presses:

```cpp
// Menu navigation
if (menu_key_set(KEY_UP) && menusub > 0)
{
    menusub--;
}
else if (menu_key_set(KEY_DOWN) && menusub < entries - 1)
{
    menusub++;
}
else if (menu_key_set(KEY_ENTER))
{
    // Handle selection based on item type
    char *p = user_io_get_confstr(menusub);
    if ((p[0] == 'O') || (p[0] == 'o'))
    {
        // Cycle through option values
        uint32_t x = user_io_status_get(p + 1, p[0] == 'o');
        x = (x + 1) % option_count;
        user_io_status_set(p + 1, x, p[0] == 'o');
    }
    else if ((p[0] == 'T') || (p[0] == 'R'))
    {
        // Trigger toggle/reset
        user_io_status_set(p + 1, 1, p[0] == 'r');
    }
    else if ((p[0] == 'F') || (p[0] == 'S'))
    {
        // Open file dialog
        open_file_dialog(p);
    }
}
```

## File Dialog Integration

### File Browser Configuration

File dialogs are configured using F and S commands:

```systemverilog
"F1,BIN;"              // File browser for BIN files
"F2,ROM,ROMX,BIN;"      // Multiple file extensions
"S0,VHD,HDF;"           // Storage mount dialog
```

### Extension Processing

The system automatically processes file extensions and adds format-specific extensions:

```cpp
void process_file_extensions(char *ext, const char *p)
{
    substrcpy(ext, p, 1);
    
    // Pad to 3-character boundaries
    while (strlen(ext) % 3) strcat(ext, " ");
    
    // Add CHD support for supported cores
    if (is_saturn() || is_pce() || is_megacd() || is_x86() || is_cdi())
    {
        if (!strcasestr(ext, "CHD"))
        {
            strcat(ext, "CHD");
        }
    }
    
    // Add ZIP support
    if (!strcasestr(ext, "ZIP"))
    {
        strcat(ext, "ZIP");
    }
}
```

### File Selection Handling

When a file is selected, the system processes it according to the dialog type:

```cpp
void handle_file_selection(int idx, const char *filename)
{
    if (user_io_file_mount(filename, idx))
    {
        // Update menu display
        char name[256];
        strncpy(name, filename, sizeof(name) - 1);
        
        // Store in recent files
        if (strlen(name) > 0)
        {
            recent_update(user_io_get_core_name(), name, 0);
        }
        
        // Update image info
        user_io_set_index(idx);
        user_io_file_info(get_extension(filename));
    }
}
```

## Conditional Display System

### Hide/Disable Flags

The conditional display system uses single-character flags to control menu item visibility:

#### Hide Flags
- `H[bit]`: Hide when status bit is **set** (1)
- `h[bit]`: Hide when status bit is **clear** (0)

#### Disable Flags
- `D[bit]`: Disable when status bit is **set** (1)
- `d[bit]`: Disable when status bit is **clear** (0)

### Multiple Conditions

Multiple conditions can be combined:

```systemverilog
"H5D3h2P1O[25],Complex Option,Off,On;"
```
This option will be:
- Hidden if status[5] = 1
- Disabled if status[3] = 1
- Hidden if status[2] = 0
- Shown on page 1

### Mask Processing

The ARM side processes these flags during menu generation:

```cpp
while ((p[0] == 'H' || p[0] == 'D' || p[0] == 'h' || p[0] == 'd') && strlen(p) > 2)
{
    uint32_t mask_bit = user_io_hd_mask(p + 1);
    int flg = (hdmask & (1 << mask_bit)) ? 1 : 0;
    
    switch (p[0])
    {
        case 'H': h |= flg; break;      // Hide if bit set
        case 'h': h |= (flg ^ 1); break; // Hide if bit clear
        case 'D': d |= flg; break;      // Disable if bit set
        case 'd': d |= (flg ^ 1); break; // Disable if bit clear
    }
    
    p += 2; // Skip flag and continue processing
}
```

### Status Menu Mask

The FPGA side can dynamically control menu visibility through the `status_menumask` signal:

```systemverilog
.status_menumask({status[7:0]})  // Use status bits 0-7 for menu control
```

This allows the core to hide/show menu items based on current operational state.

## Data Flow and Communication

### FPGA to ARM Communication

1. **Configuration String Transfer**:
   ```
   ARM                    FPGA
   ├── UIO_GET_STRING ──→ │
   │                      │ ├── Send CONF_STR
   │ ←── Configuration ───┤ │   byte by byte
   │      String Data     │ └── Terminate with 0
   ```

2. **Status Mask Request**:
   ```
   ARM                    FPGA
   ├── UIO_GET_OSDMASK ──→ │
   │ ←── Menu Mask ───────┤ └── Return 16-bit mask
   ```

### ARM to FPGA Communication

1. **Status Register Update**:
   ```
   ARM                    FPGA
   ├── UIO_SET_STATUS ───→ │
   │   + Status Data      │ ├── Update status[127:0]
   │                      │ └── Trigger change events
   ```

2. **File Transfer**:
   ```
   ARM                    FPGA
   ├── UIO_FILE_TX ──────→ │
   │   + File Data        │ ├── Receive file blocks
   │                      │ └── Write to core memory
   ```

### Synchronization and Timing

The communication is synchronized to the system clock on the FPGA side:

```systemverilog
always @(posedge clk_sys) begin
    if (reset) begin
        status <= 0;
        status_set <= 0;
    end else begin
        // Process incoming status updates
        if (status_wr) begin
            status <= status_in;
            status_set <= status_set | status_mask;
        end
        
        // Clear momentary triggers
        status_set <= status_set & ~momentary_mask;
    end
end
```

## Advanced Features

### Configuration Versioning

The version system ensures configuration compatibility:

```systemverilog
"v,2;"  // Configuration version 2
```

When the configuration version changes, all settings are reset to defaults:

```cpp
void check_config_version()
{
    int current_version = 0;
    int stored_version = 0;
    
    // Extract version from CONF_STR
    char *p = user_io_get_confstr_by_prefix("v,");
    if (p) current_version = atoi(p + 2);
    
    // Load stored version
    if (FileLoad("config_version", &stored_version, sizeof(stored_version)) && 
        stored_version == current_version)
    {
        return; // Versions match, keep current config
    }
    
    // Reset configuration to defaults
    user_io_status_reset();
    
    // Save new version
    FileSave("config_version", &current_version, sizeof(current_version));
}
```

### Dynamic Menu Reconfiguration

Some cores support dynamic menu reconfiguration by modifying the CONF_STR at runtime:

```systemverilog
reg [7:0] conf_str_mem [0:4095];
reg [11:0] conf_str_addr;

// Load different configurations based on mode
always @(posedge clk_sys) begin
    case (core_mode)
        MODE_ARCADE: load_arcade_config();
        MODE_CONSOLE: load_console_config();
        MODE_COMPUTER: load_computer_config();
    endcase
end
```

### Multi-Language Support

The system can support multiple languages by providing language-specific configuration strings:

```systemverilog
localparam CONF_STR_EN = {
    "MyCore;;",
    "O[2],TV Mode,NTSC,PAL;",
    // English configuration
};

localparam CONF_STR_DE = {
    "MyCore;;",
    "O[2],TV Modus,NTSC,PAL;",
    // German configuration
};

wire [7:0] language = status[15:8];
wire [STRLEN*8-1:0] active_conf_str = (language == 1) ? CONF_STR_DE : CONF_STR_EN;
```

### Custom Aspect Ratios

The framework supports custom aspect ratios through special placeholder syntax:

```systemverilog
"O[122:121],Aspect ratio,Original,Full Screen,[ARC1],[ARC2];"
```

The `[ARC1]` and `[ARC2]` placeholders are replaced with user-defined aspect ratio names from the MiSTer.ini configuration.

### Joystick Mapping Integration

File dialogs can trigger automatic joystick remapping:

```systemverilog
"F1,BIN,ROM;"  // File dialog that may trigger controller mapping
```

When certain file types are loaded, the system can automatically apply controller-specific button mappings stored in the gamecontroller database.

## Best Practices and Examples

### PSX Core Configuration String - Complete Analysis

The PSX core provides an excellent real-world example showcasing nearly every feature of the CONF_STR system. Here's a complete line-by-line analysis:

```systemverilog
`include "build_id.v"
parameter CONF_STR = {
```

#### Core Header and Save State Support
```systemverilog
"PSX;SS3E000000:400000;",
```
- **Core Name**: "PSX"
- **Save State Support**: `SS3E000000:400000` enables save states
  - `SS` = Save State enable flag
  - `3E000000` = Base address for save state data
  - `400000` = Size of save state data (4MB)

#### CD-ROM Management with Conditional Display
```systemverilog
"H7S1,CUECHD,Load CD;",
"h7-,Reload core for CD;",
"F1,EXE,Load Exe;",
"-;",
```
- **Line 1**: `H7` = Hide when status[7] is set (hide CD loader when CD mounted)
  - `S1` = Storage mount dialog for index 1
  - `CUECHD` = Accepts CUE and CHD file formats
- **Line 2**: `h7` = Hide when status[7] is clear (show reload message when CD mounted)
  - `-` = Separator line (shows as text)
- **Line 3**: `F1` = File dialog for index 1, EXE files (PlayStation executables)
- **Line 4**: Simple separator

#### Cheat System with Conditional Access
```systemverilog
"d6C,Cheats;",
"h6O[6],Cheats Enabled,Yes,No;",
"-;",
```
- **Line 1**: `d6` = Disable when status[6] is clear (disable cheat menu when cheats off)
  - `C` = Cheat system access
- **Line 2**: `h6` = Hide when status[6] is clear (hide cheat toggle when cheats available)
  - Status bit 6 controls cheat enable/disable
- **Line 3**: Separator

#### Memory Card Management with Status Indicators
```systemverilog
"hA-,Memcard Status: not saved;",
"HB-,Memcard Status: saved;", 
"hC-,Memcard Status: saving...;",
"RD,Save Memory Cards;",
"O[71],Save to SDCard,On Open OSD,Manual;",
"SC2,SAVMCD,Mount Memory Card 1;",
"SC3,SAVMCD,Mount Memory Card 2;",
"O[63],Automount Memory Card 1,Yes,No;",
"-;",
```
- **Lines 1-3**: Dynamic status messages using hide conditions
  - `hA` = Show "not saved" when status[10] is clear
  - `HB` = Show "saved" when status[11] is set
  - `hC` = Show "saving..." when status[12] is clear
- **Line 4**: `R` = Reset trigger for memory card save operation
- **Line 5**: Status bit 71 for save timing preference
- **Lines 6-7**: Storage mount dialogs for memory card files
- **Line 8**: Auto-mount preference for memory card 1

#### Save State Management
```systemverilog
"O[36],Savestates to SDCard,On,Off;",
"O[68],Autoincrement Slot,Off,On;",
"O[38:37],Savestate Slot,1,2,3,4;",
"RH,Save state (Alt-F1);",
"RI,Restore state (F1);",
"-;",
```
- **Line 1**: Storage location preference (bit 36)
- **Line 2**: Auto-increment slot selection (bit 68)
- **Line 3**: 2-bit slot selector (bits 37-38, 4 slots)
- **Lines 4-5**: Reset triggers for save/restore operations
- **Line 6**: Separator

#### System Configuration
```systemverilog
"O[40:39],System Type,Auto,NTSC-U,NTSC-J,PAL;",
"-;",
```
- **2-bit system type selector** (bits 39-40, 4 options)
- Auto-detection or forced region setting

#### Advanced Controller Configuration
```systemverilog
"D8O[48:45],Pad1,Dualshock,Off,Digital,Analog,GunCon,NeGcon,Wheel-NegCon,Wheel-Analog,Mouse,Justifier,SNAC-port1,Analog Joystick,Pop'n;",
"D8O[52:49],Pad2,Dualshock,Off,Digital,Analog,GunCon,NeGcon,Wheel-NegCon,Wheel-Analog,Mouse,Justifier,SNAC-port2,Analog Joystick,Pop'n;",
"D8h0O[66],SNAC MemCard,Virtual,Real;",
"D8hFO[91],NeGcon Rumble,Off,On;",
"D8h2O[9],Show Crosshair,Off,On;",
"D8h4O[31],DS Mode,L3+R3+Up/Dn | Click,L1+L2+R1+R2+Up/Dn;",
"O[57:56],Multitap,Off,Port1: 4 x Digital,Port1: 4 x Analog;",
"-;",
```
- **Lines 1-2**: `D8` = Disable when status[8] is set (disable during gameplay)
  - 4-bit controller type selectors (13 different controller types each)
- **Lines 3-6**: Additional controller options with multiple conditions
  - `h0` = Hide when SNAC disabled, `hF` = Hide when NeGcon not selected, etc.
- **Line 7**: Multitap configuration (2 bits, 3 options)

#### Multi-Page Video & Audio Settings
```systemverilog
"P1,Video & Audio;",
"P1-;",
"P1O[33:32],Aspect ratio,Original,Full Screen,[ARC1],[ARC2];",
"P1O[35:34],Scale,Normal,V-Integer,Narrower HV-Integer,Wider HV-Integer;",
"P1-;",
"DEP1O[62],Fixed HBlank,On,Off;",
"DEP1O[55],Fixed VBlank,Off,On;",
"d5P1O[4:3],Vertical Crop,Off,On(224/270),On(216/256);",
"P1O[67],Horizontal Crop,Off,On;",
"P1O[61],Black Transitions,On,Off;",
"P1O[41],Deinterlacing,Weave,Bob;",
"P1O[60],Sync 480i for HDMI,Off,On;",
"P1O[24],Rotate,Off,On;",
"P1-;",
"P1O[22],Dithering,On,Off;",
"DEP1O[84],Render 24 Bit,Off,On;",
"P1O[73],Dither 24 Bit for VGA,Off,On;",
"P1-;",
"P1O[89],480i to 480p Hack,Off,On;",
"P1O[54:53],Widescreen Hack,Off,3:2,5:3,16:9;",
"P1O[82:81],Texture Filter,Off,All Polygon,Dithered,Dith+Shaded;",
"hDP1O[87:86],Filter Strength,25%,50%,75%,100%;",
"hDP1O[83],Filter 2D Detect,Off,On;",
"P1-;",
"d1P1O[44],SPU RAM select,DDR3,SDRAM2;",
"P1O[8:7],Stereo Mix,None,25%,50%,100%;",
```
- **Page 1 Definition**: All options prefixed with `P1`
- **Multiple Conditions**: `DE` = Disable when debug enabled AND status[14] set
- **Conditional Visibility**: `d5` = Disable when status[5] clear, `hD` = Hide when status[13] set
- **Custom Aspect Ratios**: `[ARC1],[ARC2]` placeholders for user-defined ratios

#### Miscellaneous Settings Page
```systemverilog
"P2,Miscellaneous;",
"P2-;",
"P2O[16],Fastboot,Off,On;",
"P2O[42],CD Lid,Closed,Open;",
"P2O[64],Pause when OSD open,On,Off;",
"P2-;",
"P2-,(U) = unsafe -> can crash;",
"P2O[80:79],Turbo(Cheats Off),Off,Low(U),Medium(U),High(U);",
"P2O[72],Pause when CD slow,On,Off(U);",
"P2O[15],PAL 60Hz Hack,Off,On(U);",
"P2O[21],CD Fast Seek,Off,On(U);",
"P2O[77:75],CD Speed,Original,Forced 1X(U),Forced 2X(U),Hack 4X(U),Hack 6X(U),Hack 8X(U);",
"P2O[78],Limit Max CD Speed,Off,On(U);",
"P2O[85],RAM(Homebrew),2 MByte,8 MByte(U);",
"P2O[90],GPU Slowdown,Off,On(U);",
"P2-;",
"P2O[28],FPS Overlay,Off,On;",
"P2O[74],Error Overlay,Off,On;",
"P2O[59],CD Slow Overlay,Off,On;",
"h9P2O[70],CD Overlay,Read,Read+Seek;",
```
- **Page 2 Definition**: Miscellaneous options
- **Safety Warnings**: "(U)" suffix indicates unsafe options
- **3-bit CD Speed**: 6 different speed options (bits 75-77)
- **Conditional Display**: `h9` = Hide when status[9] clear

#### Debug Page (Conditionally Hidden)
```systemverilog
"h3-;",
"h3P3,Debug;",
"h3P3-;",
"h3P3O[14],DDR3 Framebuffer,Off,On;",
"h3P3O[10],DDR3 FB Color,16,24;",
"h3P3O[11],VRAMViewer,Off,On;",
"h3P3O[30],Sound,On,Off;",
"h3P3O[43],RepTimingSPUDMA,Off,On;",
"h3P3O[27],Textures,On,Off;",
"h3P3O[69],LBA Overlay,Off,On;",
"h3P3O[88],Fast CD DMA Timing,Off,On;",
"h3P3T1,Advance Pause;",
"h3P3T2,Sound IRQ Trigger;",
```
- **Entire Page Hidden**: `h3` = Hide when status[3] clear (debug mode disabled)
- **Debug Options**: Development and testing features
- **Trigger Commands**: `T1`, `T2` for debug functions

#### System Controls and Button Mapping
```systemverilog
"-   ;",
"R0,Reset;",
"J1,Triangle(NeGcon B),O(Gun Fire|NeGcon A),X(Gun B|NeGcon I),[](NeGcon II),Select,Start(Gun A),L1,R1,L2,R2,L3,R3,Savestates,Fastforward,Pause(Core),Toggle Dualshock;",
"jn,X,A,B,Y,Select,Start,L,R;",
```
- **Line 1**: Extended separator with spaces
- **Line 2**: System reset trigger
- **Line 3**: `J1` = Primary joystick button mapping with context-sensitive labels
- **Line 4**: `jn` = Alternative button names for different controller types

#### Status Information Display
```systemverilog
"I,",
"Load=DPAD Up|Save=Down|Slot=L+R,",
"Active Slot 1,",
"Active Slot 2,",
"Active Slot 3,",
"Active Slot 4,",
"Save to state 1,",
"Restore state 1,",
"Save to state 2,",
"Restore state 2,",
"Save to state 3,",
"Restore state 3,",
"Save to state 4,",
"Restore state 4,",
"Rewinding...,",
"Slot 1 Analog,",
"Slot 1 Digital,",
"Slot 2 Analog,",
"Slot 2 Digital,",
"Region Unknown->US,",
"Region JP,",
"Region US,",
"Region EU,",
"Saving Memcard,",
"Unsafe option used!;",
```
- **Information Array**: `I,` introduces status message definitions
- **Dynamic Messages**: System displays appropriate message based on core state
- **User Feedback**: Instructions, warnings, and status updates

#### Version Information
```systemverilog
"V,v",`BUILD_DATE
};
```
- **Version Display**: Shows build date from build system

### Key Patterns Demonstrated

#### 1. **Conditional Display Matrix**
```systemverilog
"H7S1,CUECHD,Load CD;",     // Hide when CD loaded
"h7-,Reload core for CD;",   // Show when CD loaded
```

#### 2. **Multi-Level Conditions**
```systemverilog
"D8h0O[66],SNAC MemCard,Virtual,Real;", // Disable during play AND hide when SNAC off
```

#### 3. **Status Bit Allocation Strategy**
- Low bits (0-31): Core functionality
- Mid bits (32-63): Video/audio settings  
- High bits (64-95): Advanced features
- Flags use single bits, multi-options use ranges

#### 4. **Page Organization**
- **Main Page**: Essential controls and file loading
- **P1**: Video & Audio settings
- **P2**: Miscellaneous and advanced options
- **P3**: Debug (hidden by default)

#### 5. **Safety Mechanisms**
```systemverilog
"P2O[80:79],Turbo(Cheats Off),Off,Low(U),Medium(U),High(U);",
```
- Clear labeling of unsafe options with "(U)" suffix
- Descriptive warnings about potential crashes

This PSX core configuration demonstrates the full power and flexibility of the MiSTer CONF_STR system, managing over 90 different configuration options across multiple pages with sophisticated conditional display logic.

### Status Register Usage

```systemverilog
// Extract configuration values
wire tv_pal = status[2];
wire [1:0] composite_mode = status[4:3];
wire [1:0] scanlines = status[6:5];
wire [1:0] scale_mode = status[8:7];
wire audio_filter = status[10];
wire [1:0] volume = status[12:11];
wire warp_mode = status[15];
wire debug_mode = status[16];

// System control
wire reset = status[0] | buttons[1];
wire cold_reset = status[1];

// Apply settings to core
always @(posedge clk_sys) begin
    if (reset || cold_reset) begin
        // Reset logic
    end else begin
        // Apply configuration
        video_pal <= tv_pal;
        video_composite <= composite_mode;
        audio_enable <= audio_filter;
        cpu_turbo <= warp_mode;
    end
end
```

### File Handling Integration

```systemverilog
// File mounting status
wire [31:0] img_mounted;
wire [31:0] img_readonly;
wire [63:0] img_size;

hps_io #(.CONF_STR(CONF_STR), .VDNUM(3)) hps_io
(
    // ... other connections
    
    .img_mounted(img_mounted),
    .img_readonly(img_readonly),
    .img_size(img_size),
    
    // File I/O
    .sd_lba(sd_lba),
    .sd_rd(sd_rd),
    .sd_wr(sd_wr),
    .sd_ack(sd_ack),
    .sd_buff_addr(sd_buff_addr),
    .sd_buff_dout(sd_buff_dout),
    .sd_buff_din(sd_buff_din),
    .sd_buff_wr(sd_buff_wr)
);

// Handle file mounting
always @(posedge clk_sys) begin
    if (img_mounted[1]) begin
        // Disk image mounted on index 1
        fdd_insert <= 1;
        fdd_size <= img_size;
        fdd_readonly <= img_readonly[1];
    end
    
    if (img_mounted[2]) begin
        // Cartridge image mounted on index 2
        cart_loaded <= 1;
        cart_size <= img_size;
    end
end
```

### Common Patterns

#### Progressive Enhancement
```systemverilog
// Basic option available to all users
"O[5],Enhancement,Off,On;",

// Advanced options only when enhancement enabled
"H5O[6],Advanced Feature 1,Off,On;",
"H5O[7],Advanced Feature 2,Off,On;",
```

#### Mode-dependent Menus
```systemverilog
// Console mode options
"h[10]O[15],Console Option,Off,On;",

// Computer mode options  
"H[10]O[16],Computer Option,Off,On;",

// Mode selector
"O[10],System Mode,Console,Computer;",
```

#### File Type Validation
```systemverilog
// ROM files with size validation
"F1,ROM,BIN;",

// Disk images with write protection
"S2,DSK,IMG;",
```

This comprehensive system provides powerful configuration management while maintaining clean separation between FPGA logic and ARM-side processing, enabling complex retro computer and console implementations with professional-grade user interfaces.