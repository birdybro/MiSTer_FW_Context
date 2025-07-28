# MiSTer HPS_IO Module - Comprehensive Technical Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture and Design](#architecture-and-design)
3. [Interface Signals and Parameters](#interface-signals-and-parameters)
4. [ARM-FPGA Communication Protocol](#arm-fpga-communication-protocol)
5. [Input Device System](#input-device-system)
6. [SD Card Emulation](#sd-card-emulation)
7. [File Transfer System](#file-transfer-system)
8. [Configuration Management](#configuration-management)
9. [PS/2 Interface](#ps2-interface)
10. [Video System Integration](#video-system-integration)
11. [Real-Time Clock and Timestamp](#real-time-clock-and-timestamp)
12. [Command Reference](#command-reference)
13. [Integration Guidelines](#integration-guidelines)
14. [Debugging and Development](#debugging-and-development)

## Overview

The `hps_io` module serves as the critical communication bridge between the ARM processor (HPS - Hard Processor System) and the FPGA fabric in the MiSTer platform. This sophisticated interface handles all aspects of system control, user input, file I/O, configuration management, and real-time data exchange.

### Key Capabilities

- **Bidirectional ARM-FPGA Communication**: High-speed data exchange via 49-bit HPS_BUS
- **Multi-Device Input Handling**: Support for 6 joysticks with digital, analog, and haptic feedback
- **Virtual SD Card Emulation**: Block-level storage interface for up to 10 virtual disks
- **File Transfer Protocol**: Efficient download/upload system for ROM images and saves
- **PS/2 Keyboard/Mouse Emulation**: Full PS/2 protocol implementation with bidirectional communication
- **Configuration String System**: Dynamic menu generation and core configuration
- **Real-Time Services**: Clock, timestamp, and timing measurement functions
- **Video Parameter Calculation**: Automatic video timing detection and reporting

### Design Philosophy

The `hps_io` module follows a command-response architecture with the ARM processor as the master controller. It provides both low-level register access and high-level protocol abstraction, enabling cores to focus on their specific functionality while leveraging common system services.

## Architecture and Design

### Module Hierarchy

```
hps_io (Top Level)
├── Parameter Configuration
├── Signal Multiplexing  
├── Command Processing Engine
├── Input Device Handlers
├── SD Card Interface
├── File Transfer Controller
├── PS/2 Device Emulation
├── Video Calculator
└── Configuration String ROM
```

### Key Design Patterns

#### 1. Parameterized Configuration
```systemverilog
module hps_io #(
    parameter CONF_STR,           // Configuration string (compile-time)
    parameter CONF_STR_BRAM = 0,  // Use BRAM for config string
    parameter PS2DIV = 0,         // PS/2 clock divider
    parameter WIDE = 0,           // 16-bit file I/O mode
    parameter VDNUM = 1,          // Number of virtual disks (1-10)
    parameter BLKSZ = 2,          // Block size: 0=128, 1=256, 2=512(default)...7=16384
    parameter PS2WE = 0,          // PS/2 write enable
    parameter STRLEN = $size(CONF_STR)>>3  // Auto-calculated string length
)
```

#### 2. Multi-Channel Communication
The module implements three distinct communication channels:
- **I/O Channel**: Standard user interface and configuration
- **File Protocol Channel**: High-speed file transfers
- **Extension Bus**: Core-specific communication extensions

#### 3. State Machine Architecture
Central command processing uses a sophisticated state machine with:
- **Command decode phase**: Determines operation type
- **Multi-byte transfer support**: Handles variable-length data
- **Error handling and recovery**: Robust operation under stress
- **Concurrent operation support**: Multiple subsystems operating simultaneously

## Interface Signals and Parameters

### Primary Communication Bus
```systemverilog
input             clk_sys,        // System clock
inout      [48:0] HPS_BUS,        // Bidirectional ARM-FPGA communication bus
```

**HPS_BUS Signal Allocation:**
- `HPS_BUS[48:46]`: System status and capability flags
- `HPS_BUS[45:38]`: Video timing signals (vs, hs, de, vs_hdmi, f1)
- `HPS_BUS[37]`: ioctl_wait signal
- `HPS_BUS[36]`: clk_sys pass-through
- `HPS_BUS[35:33]`: Channel enable signals (fp_enable, io_enable, io_strobe)
- `HPS_BUS[32]`: Data width mode indicator
- `HPS_BUS[31:16]`: Bidirectional data bus
- `HPS_BUS[15:0]`: Output data/extension bus interface

### Input Device Interfaces

#### Digital Joystick Outputs (6 controllers supported)
```systemverilog
output reg [31:0] joystick_0,     // Player 1 digital inputs
output reg [31:0] joystick_1,     // Player 2 digital inputs
output reg [31:0] joystick_2,     // Player 3 digital inputs
output reg [31:0] joystick_3,     // Player 4 digital inputs
output reg [31:0] joystick_4,     // Player 5 digital inputs
output reg [31:0] joystick_5,     // Player 6 digital inputs
```

**Joystick Bit Mapping:**
```
Bit 31-16: Extended buttons and system functions
Bit 15: Start
Bit 14: Coin/Select  
Bit 13: Menu/Guide
Bit 12: L3 (left stick click)
Bit 11: R3 (right stick click)
Bit 10: L1 (left shoulder)
Bit 9:  R1 (right shoulder)
Bit 8:  L2 (left trigger)
Bit 7:  R2 (right trigger)
Bit 6:  X/A button
Bit 5:  Y/B button
Bit 4:  B/X button
Bit 3:  A/Y button
Bit 2:  Right
Bit 1:  Left
Bit 0:  Down
Bit 31: Up (in extended mapping)
```

#### Analog Input Outputs
```systemverilog
// Left analog stick (Y: [15:8], X: [7:0], signed -127 to +127)
output reg [15:0] joystick_l_analog_0,
output reg [15:0] joystick_l_analog_1,
output reg [15:0] joystick_l_analog_2,
output reg [15:0] joystick_l_analog_3,
output reg [15:0] joystick_l_analog_4,
output reg [15:0] joystick_l_analog_5,

// Right analog stick (Y: [15:8], X: [7:0], signed -127 to +127)  
output reg [15:0] joystick_r_analog_0,
output reg [15:0] joystick_r_analog_1,
output reg [15:0] joystick_r_analog_2,
output reg [15:0] joystick_r_analog_3,
output reg [15:0] joystick_r_analog_4,
output reg [15:0] joystick_r_analog_5,
```

#### Haptic Feedback Inputs
```systemverilog
// Rumble motor control (15:8 - large motor, 7:0 - small motor)
input      [15:0] joystick_0_rumble,
input      [15:0] joystick_1_rumble,
input      [15:0] joystick_2_rumble,
input      [15:0] joystick_3_rumble,
input      [15:0] joystick_4_rumble,
input      [15:0] joystick_5_rumble,
```

#### Specialized Input Devices
```systemverilog
// Paddle controllers (0-255 range)
output reg  [7:0] paddle_0,
output reg  [7:0] paddle_1,
output reg  [7:0] paddle_2,
output reg  [7:0] paddle_3,
output reg  [7:0] paddle_4,
output reg  [7:0] paddle_5,

// Spinner/trackball ([7:0] signed delta, [8] toggle bit)
output reg  [8:0] spinner_0,
output reg  [8:0] spinner_1,
output reg  [8:0] spinner_2,
output reg  [8:0] spinner_3,
output reg  [8:0] spinner_4,
output reg  [8:0] spinner_5,
```

### PS/2 Interface Signals
```systemverilog
// PS/2 Keyboard
output            ps2_kbd_clk_out,    // Keyboard clock output
output            ps2_kbd_data_out,   // Keyboard data output
input             ps2_kbd_clk_in,     // Keyboard clock input
input             ps2_kbd_data_in,    // Keyboard data input
input       [2:0] ps2_kbd_led_status, // LED status (Caps, Num, Scroll)
input       [2:0] ps2_kbd_led_use,    // LED usage enable

// PS/2 Mouse
output            ps2_mouse_clk_out,  // Mouse clock output
output            ps2_mouse_data_out, // Mouse data output
input             ps2_mouse_clk_in,   // Mouse clock input
input             ps2_mouse_data_in,  // Mouse data input

// Alternative PS/2 interfaces
output reg [10:0] ps2_key = 0,        // [10] toggle, [9] pressed, [8] extended, [7:0] scancode
output reg [24:0] ps2_mouse = 0,      // [24] toggle, [23:16] Y, [15:8] X, [7:0] buttons
output reg [15:0] ps2_mouse_ext = 0,  // [15:8] extra buttons, [7:0] wheel
```

### System Control Outputs
```systemverilog
output      [1:0] buttons,            // Physical button states
output            forced_scandoubler,  // Force VGA scandoubler
output            direct_video,        // Direct video mode
input             video_rotated,       // Video rotation status
input             new_vmode,          // Video mode change notification
```

### Configuration and Status
```systemverilog
output reg [127:0] status,            // 128-bit configuration status
input      [127:0] status_in,         // Status input from core
input              status_set,         // Status update request
input       [15:0] status_menumask,   // Menu item visibility mask
```

### SD Card Emulation Interface
```systemverilog
// SD card configuration
output reg [VD:0] img_mounted,        // Image mount notification
output reg        img_readonly,       // Read-only mount flag
output reg [63:0] img_size,          // Image size in bytes

// Block-level SD card access
input      [31:0] sd_lba[VDNUM],     // Logical Block Address array
input       [5:0] sd_blk_cnt[VDNUM], // Block count minus 1
input      [VD:0] sd_rd,             // Read request per drive
input      [VD:0] sd_wr,             // Write request per drive
output reg [VD:0] sd_ack,            // Acknowledge per drive

// Byte-level buffer interface
output reg [AW:0] sd_buff_addr,      // Buffer address
output reg [DW:0] sd_buff_dout,      // Buffer data output
input      [DW:0] sd_buff_din[VDNUM], // Buffer data input per drive
output reg        sd_buff_wr,        // Buffer write enable
```

### File Transfer Interface
```systemverilog
// Download (ARM → FPGA)
output reg        ioctl_download = 0, // Download active flag
output reg [15:0] ioctl_index,        // File index
output reg        ioctl_wr,           // Write strobe
output reg [26:0] ioctl_addr,         // Address (incremented by 2 in WIDE mode)
output reg [DW:0] ioctl_dout,         // Data output
output reg [31:0] ioctl_file_ext,     // File extension

// Upload (FPGA → ARM)
output reg        ioctl_upload = 0,   // Upload active flag
input             ioctl_upload_req,   // Upload request
input       [7:0] ioctl_upload_index, // Upload file index
input      [DW:0] ioctl_din,          // Data input
output reg        ioctl_rd,           // Read strobe
input             ioctl_wait,         // Wait request
```

### System Information Outputs
```systemverilog
output reg [15:0] sdram_sz,           // SDRAM size configuration
output reg [64:0] RTC,                // Real-time clock data
output reg [32:0] TIMESTAMP,          // Unix timestamp
output reg  [7:0] uart_mode,          // UART configuration
output reg [31:0] uart_speed,         // UART speed
```

### Extension Interface
```systemverilog
inout      [35:0] EXT_BUS             // Extension bus for core-specific features
```

## ARM-FPGA Communication Protocol

### Protocol Stack Overview

The communication between ARM and FPGA follows a layered protocol architecture:

```
┌─────────────────────────────────────────┐
│          Application Layer              │ ← Core-specific protocols
├─────────────────────────────────────────┤
│          Session Layer                  │ ← Command/response management
├─────────────────────────────────────────┤
│          Transport Layer                │ ← Multi-byte data transfer
├─────────────────────────────────────────┤
│          Data Link Layer                │ ← SPI transaction control
├─────────────────────────────────────────┤
│          Physical Layer                 │ ← HPS_BUS electrical interface
└─────────────────────────────────────────┘
```

### Communication Channels

#### 1. Standard I/O Channel
**Enable Signal**: `io_enable = HPS_BUS[34]`
**Strobe Signal**: `io_strobe = HPS_BUS[33]`

Used for:
- Configuration updates
- Input device data
- Status synchronization
- System control

#### 2. File Protocol Channel  
**Enable Signal**: `fp_enable = HPS_BUS[35]`

Used for:
- High-speed file transfers
- ROM loading
- Save file management
- Bulk data operations

#### 3. Extension Channel
**Control Signal**: `EXT_BUS[32]`

Used for:
- Core-specific protocols
- Custom hardware interfaces
- Specialized communication needs

### Data Transfer Modes

#### Wide Mode Configuration
```systemverilog
localparam DW = (WIDE) ? 15 : 7;    // Data width: 16-bit or 8-bit
localparam AW = (WIDE) ? 12 : 13;   // Address width adjustment
```

**8-bit Mode (WIDE=0)**:
- Data: `HPS_BUS[31:24]` (8 bits)
- Buffer: 8KB (13-bit addressing)
- Address increment: 1

**16-bit Mode (WIDE=1)**:
- Data: `HPS_BUS[31:16]` (16 bits)  
- Buffer: 4KB (12-bit addressing)
- Address increment: 2

### Command Processing State Machine

```systemverilog
always@(posedge clk_sys) begin : uio_block
    reg [15:0] cmd;
    reg [MAX_W:0] byte_cnt;
    
    if(~io_enable) begin
        // Command completion processing
        cmd <= 0;
        byte_cnt <= 0;
        // Execute command-specific completion actions
    end
    else if(io_strobe) begin
        if(byte_cnt == 0) begin
            // Command decode phase
            cmd <= io_din;
            // Execute immediate command responses
        end else begin
            // Data transfer phase
            // Handle multi-byte command data
        end
        
        if(~&byte_cnt) byte_cnt <= byte_cnt + 1'd1;
    end
end
```

### Command Categories

#### Immediate Commands (Single Transaction)
- Return data immediately on command byte
- Used for status queries and simple operations
- Examples: `0x2B` (capabilities), `0x29` (status flags)

#### Multi-Byte Commands (Variable Length)
- Require additional data bytes after command
- Support large data transfers
- Examples: `0x1E` (128-bit status), `0x17` (sector write)

#### Continuous Commands (Block Transfers)
- Optimized for bulk data movement
- Use specialized state machines
- Examples: File download/upload protocols

## Input Device System

### Joystick Data Processing

The input system supports 6 simultaneous controllers with comprehensive input types:

#### Digital Input Mapping
```systemverilog
// Command 0x02: Joystick 0 (32-bit)
'h02: if(byte_cnt==1) joystick_0[15:0] <= io_din; 
      else joystick_0[31:16] <= io_din;

// Commands 0x03, 0x10-0x13: Joysticks 1-5
'h03: if(byte_cnt==1) joystick_1[15:0] <= io_din; 
      else joystick_1[31:16] <= io_din;
// ... similar for joysticks 2-5
```

#### Analog Input Processing
```systemverilog
// Command 0x1A: Left analog stick
'h1a: if(!byte_cnt[MAX_W:2]) begin
    case(byte_cnt[1:0])
        1: {pdsp_idx,stick_idx} <= io_din[7:0];
        2: case(stick_idx)
            0: joystick_l_analog_0 <= io_din;
            1: joystick_l_analog_1 <= io_din;
            // ... cases for all 6 controllers
        endcase
    endcase
end
```

#### Specialized Input Devices
```systemverilog
// Paddle support (analog dial controllers)
15: case(pdsp_idx)
    0: paddle_0 <= io_din[7:0];
    1: paddle_1 <= io_din[7:0];
    // ... cases for all 6 paddles
    
// Spinner support (trackball/rotating controllers)  
8:  spinner_0 <= {~spinner_0[8],io_din[7:0]};
9:  spinner_1 <= {~spinner_1[8],io_din[7:0]};
// ... cases for all 6 spinners
endcase
```

#### Haptic Feedback Support
```systemverilog
// Rumble motor readback commands
'h003F: io_dout <= joystick_0_rumble;
'h013F: io_dout <= joystick_1_rumble;
'h023F: io_dout <= joystick_2_rumble;
// ... for all 6 controllers
```

**Rumble Data Format**:
- Bits [15:8]: Large/primary rumble motor magnitude (0-255)
- Bits [7:0]: Small/secondary rumble motor magnitude (0-255)

### Mouse and Keyboard Integration

#### PS/2 Mouse Data Structure
```systemverilog
// Command 0x04: Mouse data
'h04: begin
    if(~&io_din[15:8] && ~ps2skip && !byte_cnt[MAX_W:2]) begin
        case(byte_cnt[1:0])
            1: ps2_mouse[7:0]   <= io_din[7:0];   // Buttons
            2: ps2_mouse[15:8]  <= io_din[7:0];   // X movement  
            3: ps2_mouse[23:16] <= io_din[7:0];   // Y movement
        endcase
        case(byte_cnt[1:0])
            1: ps2_mouse_ext[7:0]  <= {io_din[14], io_din[14:8]}; // Wheel + extra
            2: ps2_mouse_ext[11:8] <= io_din[11:8];  // Extended buttons
            3: ps2_mouse_ext[15:12]<= io_din[11:8];  // Reserved
        endcase
    end
end
```

#### PS/2 Keyboard Processing
```systemverilog
// Command 0x05: Keyboard data
'h05: begin
    if(~&io_din[15:8] & ~ps2skip) 
        ps2_key_raw[31:0] <= {ps2_key_raw[23:0], io_din[7:0]};
end

// Special key processing at command completion
if(cmd == 5 && !ps2skip) begin
    ps2_key <= {~ps2_key[10], pressed, extended, ps2_key_raw[7:0]};
    // Handle special key combinations
    if(ps2_key_raw == 'hE012E07C) ps2_key[9:0] <= 'h37C; // Print Screen pressed
    if(ps2_key_raw == 'h7CE0F012) ps2_key[9:0] <= 'h17C; // Print Screen released  
    if(ps2_key_raw == 'hF014F077) ps2_key[9:0] <= 'h377; // Pause pressed
end
```

## SD Card Emulation

### Virtual Disk Architecture

The hps_io module provides comprehensive SD card emulation supporting up to 10 virtual disk drives:

```systemverilog
parameter VDNUM = 1,                    // Number of virtual disks (1-10)
parameter BLKSZ = 2,                    // Block size (0=128, 1=256, 2=512, ..., 7=16384)
localparam VD = VDNUM-1;               // Virtual disk index range
```

### Block-Level Interface

#### Drive Selection and Round-Robin
```systemverilog
reg [3:0] sdn;     // Selected drive number
reg [3:0] sd_rrb = 0;  // Round-robin base

always_comb begin
    int n, i;
    sdn = 0;
    for(i = VDNUM - 1; i >= 0; i = i - 1) begin
        n = i + sd_rrb;
        if(n >= VDNUM) n = n - VDNUM;
        if(sd_wr[n] | sd_rd[n]) sdn = n[3:0];
    end
end
```

#### SD Card Status Reporting
```systemverilog
// Command 0x16: SD card status query
'h16: begin
    io_dout <= {1'b1, sd_blk_cnt[sdn], BLKSZ[2:0], sdn, sd_wr[sdn], sd_rd[sdn]};
    sdn_r <= sdn;  // Remember selected drive
end

// Status format breakdown:
// Bit 15:    Always 1 (valid data indicator)
// Bits 14:9: Block count minus 1 (6 bits = up to 63 blocks)  
// Bits 8:6:  Block size selector (3 bits)
// Bits 5:2:  Drive number (4 bits = up to 16 drives)
// Bit 1:     Write operation active
// Bit 0:     Read operation active
```

#### LBA (Logical Block Address) Reporting
```systemverilog
// Extended status for multi-byte reads
'h16: if(!byte_cnt[MAX_W:2]) begin
    case(byte_cnt[1:0])
        1: sd_rrb  <= (sd_rrb == VD) ? 4'd0 : (sd_rrb + 1'd1);  // Advance round-robin
        2: io_dout <= sd_lba[sdn_r][15:0];   // LBA low word
        3: io_dout <= sd_lba[sdn_r][31:16];  // LBA high word  
    endcase
end
```

### Data Transfer Operations

#### Sector Write (ARM → FPGA)
```systemverilog
// Command 0x17: Send sector data to FPGA
'h0X17: begin
    sd_buff_dout <= io_din[DW:0];  // Store data in buffer
    b_wr <= 1;                     // Trigger write state machine
end

// Write state machine with address auto-increment
sd_buff_wr <= b_wr[0];                          // Write enable
if(b_wr[2] && (~&sd_buff_addr)) 
    sd_buff_addr <= sd_buff_addr + 1'b1;        // Address increment
b_wr <= (b_wr<<1);                             // Shift register for timing
```

#### Sector Read (FPGA → ARM)
```systemverilog
// Command 0x18: Read sector data from FPGA  
'h0X18: begin
    if(~&sd_buff_addr) sd_buff_addr <= sd_buff_addr + 1'b1;  // Address increment
    io_dout <= sd_buff_din[sdn_ack];                         // Output selected drive data
end
```

#### Drive Acknowledgment
```systemverilog
// Commands 0x17/0x18: Drive operation acknowledgment
'h0X17,
'h0X18: begin
    sd_ack <= disk[VD:0];          // Set acknowledgment for selected drives
    sdn_ack <= io_din[11:8];       // Remember which drive to acknowledge
end

wire [15:0] disk = 16'd1 << io_din[11:8];  // Convert drive number to bit mask
```

### Image Mounting Protocol

#### Mount Notification
```systemverilog
// Command 0x1C: Image mount notification
'h1c: begin
    img_mounted  <= io_din[VD:0] ? io_din[VD:0] : 1'b1;  // Set mounted drives
    img_readonly <= io_din[7];                            // Read-only flag
end
```

#### Image Size Transfer
```systemverilog
// Command 0x1D: Send image size (64-bit value)
'h1d: if(byte_cnt<5) 
    img_size[{byte_cnt-1'b1, 4'b0000} +:16] <= io_din;

// Size calculation: 
// byte_cnt=1: img_size[15:0]   <= io_din;
// byte_cnt=2: img_size[31:16]  <= io_din;  
// byte_cnt=3: img_size[47:32]  <= io_din;
// byte_cnt=4: img_size[63:48]  <= io_din;
```

### Buffer Management

The SD card emulation uses a shared buffer system:

**8-bit Mode Buffer**:
- Size: 8KB (8192 bytes)
- Address range: 13 bits (0-8191)
- Suitable for sectors up to 8KB

**16-bit Mode Buffer**:
- Size: 4KB words (8KB bytes)
- Address range: 12 bits (0-4095)  
- Optimized for 16-bit data transfers

**Block Size Support**:
```systemverilog
// BLKSZ parameter mapping:
// 0 → 128 bytes   (2^7)
// 1 → 256 bytes   (2^8)  
// 2 → 512 bytes   (2^9)  ← Default
// 3 → 1024 bytes  (2^10)
// 4 → 2048 bytes  (2^11)
// 5 → 4096 bytes  (2^12)
// 6 → 8192 bytes  (2^13)
// 7 → 16384 bytes (2^14)
```

## File Transfer System

### File Transfer Protocol Overview

The hps_io module implements a sophisticated file transfer protocol for efficient ROM loading, save file management, and bulk data operations between ARM and FPGA.

### Protocol Commands

```systemverilog
localparam FIO_FILE_TX      = 8'h53;  // File transfer control
localparam FIO_FILE_TX_DAT  = 8'h54;  // File data transfer  
localparam FIO_FILE_INDEX   = 8'h55;  // File index selection
localparam FIO_FILE_INFO    = 8'h56;  // File information
```

### File Transfer State Machine

```systemverilog
always@(posedge clk_sys) begin : fio_block
    reg [15:0] cmd;
    reg  [2:0] cnt;
    reg        has_cmd;
    reg [26:0] addr;
    reg        wr;

    ioctl_rd <= 0;          // Clear read strobe
    ioctl_wr <= wr;         // Apply write strobe
    wr <= 0;                // Clear write request

    if(~fp_enable) has_cmd <= 0;  // Reset command state
    else begin
        if(io_strobe) begin
            if(!has_cmd) begin
                cmd <= io_din;      // Capture command
                has_cmd <= 1;       // Set command flag
                cnt <= 0;           // Reset byte counter
            end else begin
                // Process command data based on command type
            end
        end
    end
end
```

### Download Operation (ARM → FPGA)

#### File Information Setup
```systemverilog
// Command 0x56: File information
FIO_FILE_INFO: if(~cnt[1]) begin
    case(cnt)
        0: ioctl_file_ext[31:16] <= io_din;  // File extension high
        1: ioctl_file_ext[15:00] <= io_din;  // File extension low
    endcase
    cnt <= cnt + 1'd1;
end
```

#### File Index Selection  
```systemverilog
// Command 0x55: File index
FIO_FILE_INDEX: begin
    ioctl_index <= io_din[15:0];  // Set file index for core identification
end
```

#### Transfer Control
```systemverilog
// Command 0x53: Transfer control
FIO_FILE_TX: begin
    cnt <= cnt + 1'd1;
    case(cnt)
        0: if(io_din[7:0] == 8'hAA) begin
               // Upload mode initiation
               ioctl_addr <= 0;
               ioctl_upload <= 1;
               ioctl_rd <= 1;
           end
           else if(io_din[7:0]) begin
               // Download mode initiation
               addr <= 0;
               ioctl_download <= 1;
           end
           else begin
               // Transfer termination
               if(ioctl_download) ioctl_addr <= addr;
               ioctl_download <= 0;
               ioctl_upload <= 0;
           end

        1: begin
               ioctl_addr[15:0] <= io_din;   // Address low word
               addr[15:0] <= io_din;
           end

        2: begin
               ioctl_addr[26:16] <= io_din[10:0];  // Address high word (27-bit total)
               addr[26:16] <= io_din[10:0];
           end
    endcase
end
```

#### Data Transfer
```systemverilog
// Command 0x54: Data transfer
FIO_FILE_TX_DAT: if(ioctl_download) begin
    ioctl_addr <= addr;                              // Set target address
    ioctl_dout <= io_din[DW:0];                     // Set output data
    wr   <= 1;                                      // Trigger write
    addr <= addr + (WIDE ? 2'd2 : 2'd1);           // Increment address
end
```

### Upload Operation (FPGA → ARM)

#### Upload Request Handling
```systemverilog
// Upload request detection
old_upload_req <= ioctl_upload_req;
if(~old_upload_req & ioctl_upload_req) upload_req <= 1;

// Command 0x3C: Upload request query
'h3C: if(upload_req) begin
    io_dout <= {ioctl_upload_index, 8'd1};  // Return index and ready flag
    upload_req <= 0;                        // Clear request
end
```

#### Upload Data Transfer
```systemverilog
// Upload data handling in FIO_FILE_TX_DAT
FIO_FILE_TX_DAT: if(~ioctl_download) begin  // Upload mode
    ioctl_addr <= ioctl_addr + (WIDE ? 2'd2 : 2'd1);  // Increment read address
    fp_dout <= ioctl_din;                              // Capture input data
    ioctl_rd <= 1;                                     // Request next data
end
```

### Address and Data Width Handling

#### Address Calculation
The module supports 27-bit addressing (128MB address space):
```systemverilog
reg [26:0] ioctl_addr;  // 27-bit address (128MB range)

// Address increment based on data width
addr <= addr + (WIDE ? 2'd2 : 2'd1);
```

#### Data Width Modes
```systemverilog
// Data width configuration  
localparam DW = (WIDE) ? 15 : 7;    // 16-bit or 8-bit data

// Data assignment based on width
ioctl_dout <= io_din[DW:0];         // Extract appropriate data width
```

### File Extension Handling

The system supports file type identification through extension codes:
```systemverilog
output reg [31:0] ioctl_file_ext,   // 32-bit file extension code

// Extension encoding examples:
// ".BIN" = 0x2E42494E
// ".ROM" = 0x2E524F4D  
// ".IMG" = 0x2E494D47
```

### Wait State Support

```systemverilog
input ioctl_wait,               // Wait request from core
assign HPS_BUS[37] = ioctl_wait; // Pass wait to ARM

// Wait handling in transfer logic
if(~ioctl_wait) begin
    // Proceed with transfer
end else begin
    // Hold current state until wait is released
end
```

## Configuration Management

### Configuration String System

The configuration string system provides dynamic menu generation and core configuration through a compile-time string parameter.

#### Configuration String Structure
```systemverilog
parameter CONF_STR,                    // Configuration string (compile-time constant)
parameter CONF_STR_BRAM = 0,          // Use BRAM instead of logic for storage
parameter STRLEN = $size(CONF_STR)>>3  // Auto-calculated string length
```

#### Configuration String Access
```systemverilog
// Command 0x14: Configuration string read
'h14: if(byte_cnt <= STRLEN) io_dout[7:0] <= conf_byte;

// String byte extraction logic
wire [7:0] conf_byte;
generate
    if(CONF_STR_BRAM) begin
        // BRAM-based storage for large configurations
        confstr_rom #(CONF_STR, STRLEN) confstr_rom(
            .*, 
            .conf_addr(byte_cnt - 1'd1)
        );
    end
    else begin
        // Logic-based storage for small configurations
        assign conf_byte = CONF_STR[{(STRLEN - byte_cnt),3'b000} +:8];
    end
endgenerate
```

#### Configuration String ROM Module
```systemverilog
module confstr_rom #(parameter CONF_STR, STRLEN)
(
    input      clk_sys,
    input      [$clog2(STRLEN+1)-1:0] conf_addr,
    output reg [7:0] conf_byte
);

reg [7:0] rom[STRLEN];

initial begin
    if( CONF_STR=="" )
        $readmemh("cfgstr.hex",rom);    // Load from file if empty
    else
        for(int i = 0; i < STRLEN; i++) 
            rom[i] = CONF_STR[((STRLEN-i)*8)-1 -:8];  // Pack string into ROM
end

always @ (posedge clk_sys) conf_byte <= rom[conf_addr];
endmodule
```

### Status Management System

#### 128-Bit Status Register
```systemverilog
output reg [127:0] status,      // Current status state
input      [127:0] status_in,   // Status input from core  
input              status_set,  // Status update trigger
input       [15:0] status_menumask,  // Menu visibility control
```

#### Status Update Protocol
```systemverilog
// Status change detection
reg [127:0] status_req;
reg [3:0] stflg = 0;
reg old_status_set = 0;

old_status_set <= status_set;
if(~old_status_set & status_set) begin
    stflg <= stflg + 1'd1;      // Increment status flag
    status_req <= status_in;    // Capture new status
end
```

#### Status Transfer Commands

**ARM → FPGA Status Update**:
```systemverilog
// Command 0x1E: 128-bit status write
'h1e: if(!byte_cnt[MAX_W:4]) begin
    case(byte_cnt[3:0])
        1: status[15:00]   <= io_din;
        2: status[31:16]   <= io_din;
        3: status[47:32]   <= io_din;
        4: status[63:48]   <= io_din;
        5: status[79:64]   <= io_din;
        6: status[95:80]   <= io_din;
        7: status[111:96]  <= io_din;
        8: status[127:112] <= io_din;
    endcase
end
```

**FPGA → ARM Status Read**:
```systemverilog
// Command 0x29: Status flag and data query
'h29: if(!byte_cnt[MAX_W:4]) begin
    case(byte_cnt[3:0])
        0: io_dout <= {4'hA, stflg};        // Status flag with identifier
        1: io_dout <= status_req[15:00];    // Status data words
        2: io_dout <= status_req[31:16];
        3: io_dout <= status_req[47:32];
        4: io_dout <= status_req[63:48];
        5: io_dout <= status_req[79:64];
        6: io_dout <= status_req[95:80];
        7: io_dout <= status_req[111:96];
        8: io_dout <= status_req[127:112];
    endcase
end
```

#### Menu Mask System
```systemverilog
// Command 0x2E: Menu mask query
'h2E: if(byte_cnt == 1) io_dout <= status_menumask;
```

The menu mask controls visibility of configuration options:
- Bit N set = Menu item N visible
- Bit N clear = Menu item N hidden  
- Enables dynamic menu adaptation based on core state

### System Configuration Commands

#### Button and System Control
```systemverilog
// Command 0x01: System configuration
'h01: cfg <= io_din;

reg [15:0] cfg;
assign buttons = cfg[1:0];              // Physical button states
assign forced_scandoubler = cfg[4];     // Force VGA scandoubler  
assign direct_video = cfg[10];          // Direct video mode
```

**Configuration Bit Mapping**:
```
cfg[1:0]  : Physical button states
cfg[2]    : VGA scaler (handled in sys_top)
cfg[3]    : Composite sync (handled in sys_top)  
cfg[4]    : Forced scandoubler
cfg[5]    : Component video (handled in sys_top)
cfg[10]   : Direct video mode
cfg[15:11]: Reserved for future use
```

#### SDRAM Configuration
```systemverilog
// Command 0x31: SDRAM size configuration
'h31: if(byte_cnt == 1) sdram_sz <= io_din;

// SDRAM size encoding:
// Bit [15]:   0 = unset, 1 = set
// Bits [1:0]: 0 = none, 1 = 32MB, 2 = 64MB, 3 = 128MB
// Bit [14]:   Debug mode enable
// Bits [13:8]: Phase shift amount (debug mode)
```

#### UART Configuration  
```systemverilog
// Command 0x3B: UART configuration (3 bytes)
'h3b: if(!byte_cnt[MAX_W:2]) begin
    case(byte_cnt[1:0])
        1: tmp2 <= io_din[7:0];                    // UART mode
        2: tmp1 <= io_din;                         // Speed low word
        3: {uart_speed, uart_mode} <= {io_din, tmp1, tmp2}; // Combined assignment
    endcase
end
```

## PS/2 Interface

### PS/2 Protocol Implementation

The hps_io module provides comprehensive PS/2 keyboard and mouse emulation with full bidirectional communication support.

### PS/2 Clock Generation
```systemverilog
generate
    if(PS2DIV) begin
        reg clk_ps2;
        always @(posedge clk_sys) begin
            integer cnt;
            cnt <= cnt + 1'd1;
            if(cnt == PS2DIV) begin
                clk_ps2 <= ~clk_ps2;    // Toggle PS/2 clock
                cnt <= 0;               // Reset counter
            end
        end
    end
endgenerate
```

**Clock Frequency Calculation**:
```
PS/2 Clock = clk_sys / (2 * PS2DIV)
Example: clk_sys = 50MHz, PS2DIV = 625
PS/2 Clock = 50MHz / (2 * 625) = 40kHz
```

### PS/2 Device Module

```systemverilog
module ps2_device #(parameter PS2_FIFO_BITS=5)
(
    input        clk_sys,         // System clock
    
    input  [7:0] wdata,           // Data to send to device
    input        we,              // Write enable
    
    input        ps2_clk,         // PS/2 clock
    output reg   ps2_clk_out,     // PS/2 clock output (open-drain)
    output reg   ps2_dat_out,     // PS/2 data output (open-drain)
    output reg   tx_empty,        // Transmit buffer empty flag
    
    input        ps2_clk_in,      // PS/2 clock input
    input        ps2_dat_in,      // PS/2 data input
    
    output [8:0] rdata,           // Received data [8]=valid, [7:0]=data
    input        rd               // Read strobe
);
```

### PS/2 Transmission State Machine

```systemverilog
reg [3:0] tx_state = 0;      // Transmitter state
reg [7:0] tx_byte;           // Current transmission byte
reg       parity;            // Parity accumulator

// Transmission states:
// 0: Idle
// 1-8: Data bits 0-7  
// 9: Parity bit
// 10: Stop bit
// 11: Complete

if(tx_state == 0) begin
    // Wait for data and bus idle
    if(c2 && c1 && d1 && wptr != rptr) begin
        timeout <= timeout - 1'd1;
        if(!timeout) begin
            tx_byte <= fifo[rptr];          // Load data byte
            rptr <= rptr + 1'd1;            // Advance read pointer
            parity <= 1;                    // Initialize odd parity
            tx_state <= 1;                  // Start transmission
            ps2_dat_out <= 0;               // Send start bit
        end
    end
end else begin
    // Data bit transmission (states 1-8)
    if((tx_state >= 1)&&(tx_state < 9)) begin
        ps2_dat_out <= tx_byte[0];          // Send LSB first
        tx_byte[6:0] <= tx_byte[7:1];       // Shift right
        if(tx_byte[0]) parity <= !parity;   // Accumulate parity
    end
    
    // Parity bit (state 9)
    if(tx_state == 9) ps2_dat_out <= parity;
    
    // Stop bit (state 10)  
    if(tx_state == 10) ps2_dat_out <= 1;
    
    // Advance state
    if(tx_state < 11) tx_state <= tx_state + 1'd1;
        else tx_state <= 0;
end
```

### PS/2 Reception State Machine

```systemverilog
reg [2:0] rx_state = 0;      // Receiver state
reg [3:0] rx_cnt;            // Bit counter
reg [7:0] data;              // Received data byte

// Reception states:
// 0: Idle (waiting for start bit)
// 1: Start bit detected
// 2: Data bits (8 bits)  
// 3: Parity bit
// 4: Stop bit

// Start bit detection (falling edge on data while clock high)
if(!rx_state && !tx_state && ~c2 && c1 && ~d1) begin
    rx_state <= rx_state + 1'b1;    // Enter reception mode
    ps2_dat_out <= 1;               // Release data line
end

// Clock edge processing
if(~old_clk & ps2_clk) begin
    if(rx_state) begin
        case(rx_state)
            1: begin
                rx_state <= rx_state + 1'b1;    // Move to data phase
                rx_cnt <= 0;                    // Reset bit counter
            end
            
            2: begin
                if(rx_cnt <= 7) 
                    data <= {d1, data[7:1]};    // Shift in data (LSB first)
                else 
                    rx_state <= rx_state + 1'b1; // Move to parity
                rx_cnt <= rx_cnt + 1'b1;
            end
            
            3: if(d1) begin                      // Check parity (should be high for odd parity)
                rx_state <= rx_state + 1'b1;
                ps2_dat_out <= 0;               // Send ACK
            end
            
            4: begin
                ps2_dat_out <= 1;               // Release data line
                has_data <= 1;                 // Signal data ready
                rx_state <= 0;                 // Return to idle
                rptr <= 0;                     // Reset FIFO pointers
                wptr <= 0;
            end
        endcase
    end
end
```

### Keyboard Interface Integration

#### Keyboard Data Processing
```systemverilog
// ARM to FPGA keyboard data
reg  [7:0] kbd_data;
reg        kbd_we;
wire [8:0] kbd_data_host;  // [8]=valid, [7:0]=data
reg        kbd_rd;

// Command 0x05: Keyboard data reception
'h05: begin
    if(&io_din[15:8]) ps2skip <= 1;               // Skip marker detected
    if(~&io_din[15:8] & ~ps2skip) 
        ps2_key_raw[31:0] <= {ps2_key_raw[23:0], io_din[7:0]}; // Shift in scancode
    if(PS2DIV) begin
        kbd_data <= io_din[7:0];                  // Store for PS/2 device
        kbd_we <= 1;                              // Trigger write
    end
end
```

#### Special Key Handling
```systemverilog
// Process special key sequences at command completion
if(cmd == 5 && !ps2skip) begin
    ps2_key <= {~ps2_key[10], pressed, extended, ps2_key_raw[7:0]};
    
    // Special key sequence handling
    if(ps2_key_raw == 'hE012E07C) ps2_key[9:0] <= 'h37C; // Print Screen pressed
    if(ps2_key_raw == 'h7CE0F012) ps2_key[9:0] <= 'h17C; // Print Screen released
    if(ps2_key_raw == 'hF014F077) ps2_key[9:0] <= 'h377; // Pause pressed
end

// Key state extraction
wire pressed  = (ps2_key_raw[15:8] != 8'hf0);              // Not a break code
wire extended = (~pressed ? (ps2_key_raw[23:16] == 8'he0) : // Extended key check
                           (ps2_key_raw[15:8] == 8'he0));
```

#### LED Status Management
```systemverilog
// Command 0x1F: Keyboard LED status query
'h1f: io_dout <= {|PS2WE, 2'b01, 
                  ps2_kbd_led_status[2], ps2_kbd_led_use[2],  // Scroll Lock
                  ps2_kbd_led_status[1], ps2_kbd_led_use[1],  // Num Lock
                  ps2_kbd_led_status[0], ps2_kbd_led_use[0]}; // Caps Lock

// LED bit format:
// Bit 7: PS/2 write enable capability
// Bits 6:5: Always 01 (identifier)
// Bit 4: Scroll Lock status
// Bit 3: Scroll Lock usage enable  
// Bit 2: Num Lock status
// Bit 1: Num Lock usage enable
// Bit 0: Caps Lock status
// Bit -1: Caps Lock usage enable (bit order adjusted)
```

### Mouse Interface Integration

#### Mouse Data Processing
```systemverilog
// Command 0x04: Mouse data reception
'h04: begin
    if(PS2DIV) begin
        mouse_data <= io_din[7:0];    // Store for PS/2 device
        mouse_we   <= 1;              // Trigger write
    end
    if(&io_din[15:8]) ps2skip <= 1;   // Skip marker
    if(~&io_din[15:8] && ~ps2skip && !byte_cnt[MAX_W:2]) begin
        case(byte_cnt[1:0])
            1: ps2_mouse[7:0]   <= io_din[7:0];   // Button states
            2: ps2_mouse[15:8]  <= io_din[7:0];   // X movement
            3: ps2_mouse[23:16] <= io_din[7:0];   // Y movement
        endcase
        case(byte_cnt[1:0])
            1: ps2_mouse_ext[7:0]  <= {io_din[14], io_din[14:8]}; // Wheel data
            2: ps2_mouse_ext[11:8] <= io_din[11:8];  // Extended buttons
            3: ps2_mouse_ext[15:12]<= io_din[11:8];  // Reserved
        endcase
    end
end

// Mouse event toggle (command completion)
if(cmd == 4 && !ps2skip) ps2_mouse[24] <= ~ps2_mouse[24];
```

#### Bidirectional PS/2 Communication
```systemverilog
// Command 0x21: PS/2 host communication
'h21: if(PS2DIV) begin
    if(byte_cnt == 1) begin
        io_dout <= kbd_data_host;    // Read keyboard data from host
        kbd_rd <= 1;                 // Trigger read
    end
    else if(byte_cnt == 2) begin
        io_dout <= mouse_data_host;  // Read mouse data from host
        mouse_rd <= 1;               // Trigger read
    end
end
```

## Video System Integration

### Video Calculation Module

The video calculation module provides comprehensive video timing analysis and parameter extraction for the ARM side to understand the current video mode.

```systemverilog
module video_calc
(
    input clk_100,                    // 100MHz reference clock
    input clk_vid,                    // Video clock
    input clk_sys,                    // System clock
    
    input ce_pix,                     // Pixel clock enable
    input de,                         // Data enable
    input hs,                         // Horizontal sync
    input vs,                         // Vertical sync
    input vs_hdmi,                    // HDMI vertical sync
    input f1,                         // Field 1 (interlaced video)
    input new_vmode,                  // Video mode change notification
    input video_rotated,              // Video rotation status
    
    input       [4:0] par_num,        // Parameter number to read
    output reg [15:0] dout            // Parameter data output
);
```

### Video Parameter Extraction

#### Parameter Reading Interface
```systemverilog
// Command 0x23: Video parameters
'h23: if(!byte_cnt[MAX_W:5]) io_dout <= vc_dout;

// Video calculator parameter selection
always @(posedge clk_sys) begin
    case(par_num)
        1: dout <= {video_rotated, |vid_int, vid_nres};      // Status and resolution change
        2: dout <= vid_hcnt[15:0];       // Horizontal count low
        3: dout <= vid_hcnt[31:16];      // Horizontal count high  
        4: dout <= vid_vcnt[15:0];       // Vertical count low
        5: dout <= vid_vcnt[31:16];      // Vertical count high
        6: dout <= vid_htime[15:0];      // Horizontal time low
        7: dout <= vid_htime[31:16];     // Horizontal time high
        8: dout <= vid_vtime[15:0];      // Vertical time low
        9: dout <= vid_vtime[31:16];     // Vertical time high
        10: dout <= vid_pix[15:0];       // Pixel count low
        11: dout <= vid_pix[31:16];      // Pixel count high
        12: dout <= vid_vtime_hdmi[15:0];  // HDMI vertical time low
        13: dout <= vid_vtime_hdmi[31:16]; // HDMI vertical time high
        14: dout <= vid_ccnt[15:0];      // Color count low
        15: dout <= vid_ccnt[31:16];     // Color count high
        16: dout <= vid_pixrep;          // Pixel repetition
        17: dout <= vid_de_h;            // Horizontal data enable
        18: dout <= vid_de_v;            // Vertical data enable
        default dout <= 0;
    endcase
end
```

### Video Timing Measurement

#### Video Clock Domain Analysis
```systemverilog
reg [31:0] vid_hcnt = 0;     // Horizontal pixel count
reg [31:0] vid_vcnt = 0;     // Vertical line count  
reg [31:0] vid_ccnt = 0;     // Color/active pixel count
reg  [7:0] vid_nres = 0;     // Resolution change counter
reg  [1:0] vid_int  = 0;     // Interlace detection
reg  [7:0] vid_pixrep;       // Pixel repetition factor
reg [15:0] vid_de_h;         // Horizontal data enable count
reg  [7:0] vid_de_v;         // Vertical data enable count

always @(posedge clk_vid) begin
    integer hcnt, vcnt, ccnt;
    reg [7:0] pcnt;                    // Pixel repetition counter
    reg [7:0] de_v;                    // Data enable vertical counter
    reg [15:0] de_h;                   // Data enable horizontal counter
    reg old_vs = 0, old_hs = 0, old_de = 0, old_vmode = 0;
    reg [3:0] resto = 0;               // Resolution timeout counter
    reg calch = 0;                     // Calculate horizontal enable

    // Color pixel counting (when calculating and data enable active)
    if(calch & de) ccnt <= ccnt + 1;
    pcnt <= pcnt + 1'd1;

    // Horizontal data enable measurement
    old_hs_vclk <= hs;
    de_h <= de_h + 1'd1;
    if(old_hs_vclk & ~hs) de_h <= 1;   // Reset on horizontal sync
    
    old_de_vclk <= de;
    if(calch & ~old_de_vclk & de) vid_de_h <= de_h;  // Capture on data enable start

    if(ce_pix) begin
        old_vs <= vs;
        old_hs <= hs;
        old_de <= de;
        old_de1 <= old_de;
        pcnt <= 1;

        // Line counting (active when not in vertical sync and data enable starts)
        if(~vs & ~old_de & de) vcnt <= vcnt + 1;
        
        // Horizontal pixel counting (when calculating and data enable active)
        if(calch & de) hcnt <= hcnt + 1;
        if(old_de & ~de) calch <= 0;                    // Stop calculating on data enable end
        if(~old_de1 & old_de) vid_pixrep <= pcnt;       // Capture pixel repetition
        if(old_hs & ~hs) de_v <= de_v + 1'd1;           // Vertical data enable counting
        if(calch & ~old_de & de) vid_de_v <= de_v;      // Capture vertical data enable

        // Frame processing (on vertical sync falling edge)
        if(old_vs & ~vs) begin
            vid_int <= {vid_int[0],f1};                 // Track interlace state
            if(~f1) begin                               // Process on even field
                if(hcnt && vcnt) begin
                    old_vmode <= new_vmode;
                    
                    // Resolution change detection with timeout
                    if(resto) resto <= resto + 1'd1;
                    if(vid_hcnt != hcnt || vid_vcnt != vcnt || old_vmode != new_vmode) 
                        resto <= 1;
                    if(&resto) vid_nres <= vid_nres + 1'd1;  // Increment change counter
                    
                    // Update video parameters
                    vid_hcnt <= hcnt;
                    vid_vcnt <= vcnt;
                    vid_ccnt <= ccnt;
                end
                vcnt <= 0;
                hcnt <= 0;
                ccnt <= 0;
                calch <= 1;                             // Enable calculation
                de_v <= 0;
            end
        end
    end
end
```

#### 100MHz Reference Domain Analysis
```systemverilog
reg [31:0] vid_htime = 0;    // Horizontal timing (100MHz clocks)
reg [31:0] vid_vtime = 0;    // Vertical timing (100MHz clocks)
reg [31:0] vid_pix = 0;      // Pixel count per frame

always @(posedge clk_100) begin
    integer vtime, htime, hcnt;
    reg old_vs, old_hs, old_vs2, old_hs2, old_de, old_de2;
    reg calch = 0;

    // Sync edge detection
    old_vs <= vs;
    old_hs <= hs;
    old_vs2 <= old_vs;
    old_hs2 <= old_hs;

    // Timing counters
    vtime <= vtime + 1'd1;
    htime <= htime + 1'd1;

    // Vertical timing measurement
    if(~old_vs2 & old_vs) begin             // Vertical sync rising edge
        vid_pix <= hcnt;                    // Capture pixels per line
        vid_vtime <= vtime;                 // Capture vertical timing
        vtime <= 0;                         // Reset vertical timer
        hcnt <= 0;                          // Reset horizontal pixel count
    end

    if(old_vs2 & ~old_vs) calch <= 1;       // Enable calculation on falling edge

    // Horizontal timing measurement  
    if(~old_hs2 & old_hs) begin             // Horizontal sync rising edge
        vid_htime <= htime;                 // Capture horizontal timing
        htime <= 0;                         // Reset horizontal timer
    end

    // Pixel counting during active video
    old_de   <= de;
    old_de2  <= old_de;
    if(calch & old_de) hcnt <= hcnt + 1;    // Count pixels during data enable
    if(old_de2 & ~old_de) calch <= 0;       // Stop counting on data enable end
end
```

#### HDMI Timing Analysis
```systemverilog
reg [31:0] vid_vtime_hdmi;   // HDMI vertical timing

always @(posedge clk_100) begin
    integer vtime;
    reg old_vs, old_vs2;

    old_vs <= vs_hdmi;
    old_vs2 <= old_vs;
    vtime <= vtime + 1'd1;

    if(~old_vs2 & old_vs) begin             // HDMI vertical sync rising edge
        vid_vtime_hdmi <= vtime;            // Capture HDMI vertical timing
        vtime <= 0;                         // Reset timer
    end
end
```

### Video Mode Change Detection

The system provides automatic detection of video mode changes:

```systemverilog
input new_vmode,             // Video mode change notification from core
input video_rotated,         // Video rotation status

// Video mode change integration
old_vmode <= new_vmode;
if(vid_hcnt != hcnt || vid_vcnt != vcnt || old_vmode != new_vmode) 
    resto <= 1;              // Trigger resolution timeout

// Resolution change reporting
if(&resto) vid_nres <= vid_nres + 1'd1;  // Increment change counter for ARM polling
```

The ARM side can monitor the `vid_nres` counter to detect when video parameters have stabilized after a mode change, ensuring accurate timing measurements.

## Real-Time Clock and Timestamp

### RTC Data Structure

The hps_io module provides comprehensive real-time clock support compatible with the MSM6242B RTC layout:

```systemverilog
output reg [64:0] RTC,        // Real-time clock data (65-bit)
output reg [32:0] TIMESTAMP,  // Unix timestamp (33-bit)
```

### RTC Format (MSM6242B Layout)

The RTC data follows the MSM6242B real-time clock IC format used in many retro computers:

```
RTC[63:60] : Second (tens)      - BCD format (0-5)
RTC[59:56] : Second (units)     - BCD format (0-9)
RTC[55:52] : Minute (tens)      - BCD format (0-5)  
RTC[51:48] : Minute (units)     - BCD format (0-9)
RTC[47:44] : Hour (tens)        - BCD format (0-2)
RTC[43:40] : Hour (units)       - BCD format (0-9)
RTC[39:36] : Day (tens)         - BCD format (0-3)
RTC[35:32] : Day (units)        - BCD format (0-9)
RTC[31:28] : Month (tens)       - BCD format (0-1)
RTC[27:24] : Month (units)      - BCD format (0-9)
RTC[23:20] : Year (tens)        - BCD format (0-9)
RTC[19:16] : Year (units)       - BCD format (0-9)
RTC[15:12] : Weekday            - BCD format (0-6, 0=Sunday)
RTC[11:8]  : Reserved/Control
RTC[7:4]   : Reserved/Control  
RTC[3:0]   : Reserved/Control
RTC[64]    : Update toggle bit  - Toggles on each update
```

### RTC Update Protocol

#### ARM to FPGA RTC Transfer
```systemverilog
// Command 0x22: RTC data transfer
'h22: RTC[(byte_cnt-6'd1)<<4 +:16] <= io_din;

// Transfer sequence:
// byte_cnt=1: RTC[15:0]   <= io_din;   // Control and weekday
// byte_cnt=2: RTC[31:16]  <= io_din;   // Day and month  
// byte_cnt=3: RTC[47:32]  <= io_din;   // Hour and minute
// byte_cnt=4: RTC[63:48]  <= io_din;   // Second data
```

#### RTC Update Completion
```systemverilog
// RTC update notification (command completion)
if(cmd == 'h22) RTC[64] <= ~RTC[64];    // Toggle update bit
```

The toggle bit (RTC[64]) allows cores to detect when new RTC data is available without constantly polling all RTC registers.

### Unix Timestamp Support

#### Timestamp Data Structure
```systemverilog
output reg [32:0] TIMESTAMP,  // Seconds since 1970-01-01 00:00:00 UTC
```

The 33-bit timestamp supports dates beyond the year 2038 (standard 32-bit Unix timestamp limit):
- Standard range: 1970-2038 (32 bits)
- Extended range: 1970-2106 (33 bits)

#### Timestamp Update Protocol
```systemverilog
// Command 0x24: Timestamp transfer
'h24: TIMESTAMP[(byte_cnt-6'd1)<<4 +:16] <= io_din;

// Transfer sequence:
// byte_cnt=1: TIMESTAMP[15:0]  <= io_din;   // Seconds low word
// byte_cnt=2: TIMESTAMP[31:16] <= io_din;   // Seconds high word
// byte_cnt=3: TIMESTAMP[32]    <= io_din[0]; // Extended bit

// Timestamp update completion
if(cmd == 'h24) TIMESTAMP[32] <= ~TIMESTAMP[32];  // Toggle update bit
```

### Time Conversion Utilities

#### BCD Encoding
For cores that need to convert binary time values to BCD for RTC compatibility:

```verilog
function [7:0] bin_to_bcd(input [7:0] binary);
    bin_to_bcd = (binary / 10) * 16 + (binary % 10);
endfunction

// Example usage:
wire [7:0] seconds_bcd = bin_to_bcd(seconds_binary);
wire [7:0] minutes_bcd = bin_to_bcd(minutes_binary);
wire [7:0] hours_bcd   = bin_to_bcd(hours_binary);
```

#### Weekday Calculation
MSM6242B weekday encoding (0-6, where 0=Sunday):

```verilog
function [2:0] calc_weekday(input [7:0] day, input [7:0] month, input [15:0] year);
    // Simplified weekday calculation (Zeller's congruence)
    reg [15:0] adj_year;
    reg [7:0] adj_month;
    reg [15:0] q;
    
    adj_year = (month < 3) ? year - 1 : year;
    adj_month = (month < 3) ? month + 12 : month;
    q = day + ((13 * (adj_month + 1)) / 5) + adj_year + (adj_year / 4) - (adj_year / 100) + (adj_year / 400);
    calc_weekday = q % 7;  // 0=Saturday, adjust as needed
endfunction
```

### RTC Integration Examples

#### Basic RTC Update (from ARM)
```c
// ARM side RTC update example
void update_rtc(struct tm *time_info) {
    uint16_t rtc_data[4];
    
    // Pack BCD data
    rtc_data[0] = (time_info->tm_wday << 12) | 0x000;  // Weekday + control
    rtc_data[1] = (bin_to_bcd(time_info->tm_mday) << 8) | 
                  bin_to_bcd(time_info->tm_mon + 1);   // Day + month
    rtc_data[2] = (bin_to_bcd(time_info->tm_hour) << 8) | 
                  bin_to_bcd(time_info->tm_min);       // Hour + minute  
    rtc_data[3] = bin_to_bcd(time_info->tm_sec);       // Seconds
    
    // Send to FPGA
    spi_uio_cmd_cont(UIO_RTC);
    for(int i = 0; i < 4; i++) {
        spi_uio_cmd_cont(rtc_data[i]);
    }
    spi_uio_cmd(0);  // End command
}
```

#### Core RTC Reading
```systemverilog
// Core side RTC usage example
reg [7:0] rtc_seconds, rtc_minutes, rtc_hours;
reg [7:0] rtc_day, rtc_month;
reg [2:0] rtc_weekday;
reg       rtc_updated;

always @(posedge clk) begin
    reg old_rtc_toggle;
    
    old_rtc_toggle <= RTC[64];
    rtc_updated <= (old_rtc_toggle != RTC[64]);  // Detect RTC update
    
    if(rtc_updated) begin
        // Extract BCD time data
        rtc_seconds  <= {RTC[63:60], RTC[59:56]};    // Tens + units
        rtc_minutes  <= {RTC[55:52], RTC[51:48]};
        rtc_hours    <= {RTC[47:44], RTC[43:40]};
        rtc_day      <= {RTC[39:36], RTC[35:32]};
        rtc_month    <= {RTC[31:28], RTC[27:24]};
        rtc_weekday  <= RTC[15:12];
    end
end
```

## Command Reference

### Command Categories and Encoding

The hps_io module implements a comprehensive command set organized by functional categories. Commands use 16-bit encoding with the following structure:

```
Bits [15:12]: Command category
Bits [11:8]:  Subcategory/device index  
Bits [7:0]:   Specific command
```

### Input Device Commands

#### Joystick Commands
```systemverilog
0x02: Joystick 0 digital inputs (32-bit)
      Byte 1: joystick_0[15:0]  - Low word
      Byte 2: joystick_0[31:16] - High word

0x03: Joystick 1 digital inputs (32-bit)
      Format identical to 0x02

0x10: Joystick 2 digital inputs (32-bit)
0x11: Joystick 3 digital inputs (32-bit)  
0x12: Joystick 4 digital inputs (32-bit)
0x13: Joystick 5 digital inputs (32-bit)

0x1A: Left analog stick data
      Byte 1: {paddle_index[7:4], stick_index[3:0]}
      Byte 2: Analog data (Y[15:8], X[7:0])
      
      stick_index mapping:
      0-5: Joysticks 0-5 left analog
      15: Special devices (paddle/spinner)
      
      paddle_index mapping (when stick_index=15):
      0-5: Paddles 0-5 (byte 2 = paddle value[7:0])
      8-13: Spinners 0-5 (byte 2 = delta[7:0], toggle[8])

0x3D: Right analog stick data
      Byte 1: stick_index[3:0]
      Byte 2: Analog data (Y[15:8], X[7:0])
      
      stick_index mapping:
      0-5: Joysticks 0-5 right analog
```

#### Rumble Feedback Query Commands
```systemverilog
0x003F: Joystick 0 rumble query - Returns joystick_0_rumble[15:0]
0x013F: Joystick 1 rumble query - Returns joystick_1_rumble[15:0]
0x023F: Joystick 2 rumble query - Returns joystick_2_rumble[15:0]
0x033F: Joystick 3 rumble query - Returns joystick_3_rumble[15:0]
0x043F: Joystick 4 rumble query - Returns joystick_4_rumble[15:0]
0x053F: Joystick 5 rumble query - Returns joystick_5_rumble[15:0]

Rumble data format:
Bits [15:8]: Large motor magnitude (0-255)
Bits [7:0]:  Small motor magnitude (0-255)
```

#### Mouse and Keyboard Commands
```systemverilog
0x04: PS/2 Mouse data transfer
      Byte 1: Button state and flags
      Byte 2: X movement (-128 to +127)
      Byte 3: Y movement (-128 to +127)
      
      Extended data (ps2_mouse_ext):
      Byte 1: Wheel data and extended buttons
      Byte 2: Additional extended buttons
      Byte 3: Reserved for future use

0x05: PS/2 Keyboard scancode transfer
      Multi-byte: Accumulates scancodes in ps2_key_raw[31:0]
      Processes special sequences:
      - E0 12 E0 7C: Print Screen press
      - 7C E0 F0 12: Print Screen release  
      - F0 14 F0 77: Pause key press

0x1F: Keyboard LED status query
      Returns: {PS2WE_flag, 2'b01, scroll_status, scroll_use, 
               num_status, num_use, caps_status, caps_use}

0x21: PS/2 bidirectional communication (when PS2DIV > 0)
      Byte 1: Keyboard data from host device
      Byte 2: Mouse data from host device
```

### System Configuration Commands

#### Basic System Control
```systemverilog
0x01: System configuration register
      cfg[15:0] controls:
      - cfg[1:0]: Physical button states
      - cfg[4]: Forced scandoubler enable
      - cfg[10]: Direct video mode

0x2B: System capabilities query
      Returns: {HPS_BUS[48:46], capability_flags[3:0]}
      
      Capability flags:
      Bit 3: Adaptive sync support (if not MISTER_DISABLE_ADAPTIVE)
      Bits 2:1: Always "11" 
      Bit 0: Always "0"

0x2F: Core feature query
      Returns: 1 (indicates core supports this query)

0x31: SDRAM configuration
      Byte 1: SDRAM size and debug settings
      
      Bit [15]: Configuration valid flag
      Bits [1:0]: Size (0=none, 1=32MB, 2=64MB, 3=128MB)
      Bit [14]: Debug mode enable
      Bits [13:8]: Debug phase shift amount

0x32: Gamma enable control  
      Byte 1: gamma_en <= io_din[0]

0x33: Gamma table programming
      Multi-byte cycle:
      Byte 1: Address high[7:0], Data[7:0]
      Byte 2: Address high[7:0], Data[7:0]  
      Byte 3: Address high[7:0], Data[7:0]
      (Auto-cycles after 3 bytes)

0x36: Info request response
      Returns: info_n[7:0] (clears after read)

0x39: Extended feature query
      Returns: 1 (indicates support)

0x3B: UART configuration (3 bytes)
      Byte 1: UART mode[7:0]
      Byte 2: UART speed[15:0]  
      Byte 3: UART speed[31:16]

0x3E: Shadow mask support query
      Returns: 1 (indicates shadow mask support)
```

### Status Management Commands

```systemverilog
0x1E: 128-bit status register write
      8 bytes: status[127:0] in 16-bit words
      Byte 1: status[15:0]
      Byte 2: status[31:16]
      ...
      Byte 8: status[127:112]

0x29: Status change query and read
      Byte 0: Returns {4'hA, stflg[3:0]} - Status change flag
      Bytes 1-8: Returns status_req[127:0] - Pending status data

0x2E: Menu mask query
      Byte 1: Returns status_menumask[15:0]
      
      Menu mask controls visibility:
      Bit N = 1: Menu item N visible
      Bit N = 0: Menu item N hidden
```

### Configuration String Commands

```systemverilog
0x14: Configuration string read
      Multi-byte: Returns CONF_STR bytes sequentially
      byte_cnt=1: Returns CONF_STR byte 0
      byte_cnt=2: Returns CONF_STR byte 1
      ...
      byte_cnt=N: Returns CONF_STR byte N-1
      
      String format: Null-terminated ASCII
      Special sequences:
      - "O[start:end],name,option1,option2,...": Multi-bit option
      - "T[bit],name": Toggle option
      - "-;": Menu separator
      - "R[bit],name": Reset option
```

### SD Card Emulation Commands

#### Status and Control
```systemverilog
0x16: SD card status query
      Immediate return: {1'b1, sd_blk_cnt[5:0], BLKSZ[2:0], 
                        sdn[3:0], sd_wr, sd_rd}
      
      Multi-byte read:
      Byte 1: Advances round-robin counter
      Byte 2: Returns sd_lba[sdn][15:0]
      Byte 3: Returns sd_lba[sdn][31:16]

0x17: SD card sector write (ARM → FPGA)
      Multi-byte: Transfers sector data to buffer
      Each byte: sd_buff_dout <= io_din[DW:0]
      Triggers: b_wr state machine for timing

0x18: SD card sector read (FPGA → ARM) 
      Multi-byte: Reads sector data from buffer
      Each byte: Returns sd_buff_din[sdn_ack]
      Auto-increments: sd_buff_addr
```

#### Image Management  
```systemverilog
0x1C: Image mount notification
      Byte 1: img_mounted[VD:0] - Drive mount flags
              img_readonly - Read-only flag (bit 7)

0x1D: Image size transfer (64-bit)
      Byte 1: img_size[15:0]
      Byte 2: img_size[31:16]
      Byte 3: img_size[47:32]  
      Byte 4: img_size[63:48]
```

### File Transfer Commands

```systemverilog
0x53: File transfer control (FIO_FILE_TX)
      3-byte sequence:
      Byte 0: Control byte
              0xAA = Upload mode start
              0x01 = Download mode start  
              0x00 = Transfer end
      Byte 1: Address[15:0]
      Byte 2: Address[26:16]

0x54: File data transfer (FIO_FILE_TX_DAT)
      Download: ioctl_dout <= io_din, increment address
      Upload: Returns ioctl_din, increment address

0x55: File index selection (FIO_FILE_INDEX)
      Byte 1: ioctl_index[15:0]

0x56: File information (FIO_FILE_INFO)  
      Byte 1: ioctl_file_ext[31:16]
      Byte 2: ioctl_file_ext[15:0]

0x3C: Upload request query
      Returns: {ioctl_upload_index[15:8], ready_flag[7:0]}
      Clears upload_req flag after read
```

### Timing and RTC Commands

```systemverilog
0x22: Real-time clock data transfer
      4 bytes: RTC data in MSM6242B format
      Byte 1: RTC[15:0]   - Control and weekday
      Byte 2: RTC[31:16]  - Day and month
      Byte 3: RTC[47:32]  - Hour and minute
      Byte 4: RTC[63:48]  - Seconds
      Completion: Toggles RTC[64] update bit

0x23: Video parameter query
      Multi-byte: Returns video_calc parameters
      Uses par_num[4:0] to select parameter:
      1: Status flags
      2-3: Horizontal count (32-bit)
      4-5: Vertical count (32-bit)
      6-7: Horizontal timing (32-bit)
      8-9: Vertical timing (32-bit)
      10-11: Pixel count (32-bit)
      12-13: HDMI vertical timing (32-bit)
      14-15: Color count (32-bit)
      16: Pixel repetition
      17: Horizontal data enable
      18: Vertical data enable

0x24: Unix timestamp transfer
      3 bytes: TIMESTAMP data
      Byte 1: TIMESTAMP[15:0]
      Byte 2: TIMESTAMP[31:16]  
      Byte 3: TIMESTAMP[32] (extended bit)
      Completion: Toggles TIMESTAMP[32] update bit
```

### Command Processing State Machine

```systemverilog
// Command decode and execution pattern
always@(posedge clk_sys) begin
    if(~io_enable) begin
        // Command completion processing
        case(cmd)
            'h04: if(~ps2skip) ps2_mouse[24] <= ~ps2_mouse[24];
            'h05: if(~ps2skip) /* process keyboard */;
            'h22: RTC[64] <= ~RTC[64];
            'h24: TIMESTAMP[32] <= ~TIMESTAMP[32];
        endcase
        cmd <= 0;
        byte_cnt <= 0;
    end
    else if(io_strobe) begin
        if(byte_cnt == 0) begin
            cmd <= io_din;  // Command capture
            // Immediate response commands
            case(io_din)
                'h16: io_dout <= sd_status;
                'h2B: io_dout <= capabilities;
                'h29: io_dout <= {4'hA, stflg};
                // ... other immediate commands
            endcase
        end else begin
            // Multi-byte command processing
            case(cmd)
                'h02: /* joystick 0 */;
                'h1E: /* status write */;
                'h22: /* RTC write */;
                // ... other multi-byte commands
            endcase
        end
        if(~&byte_cnt) byte_cnt <= byte_cnt + 1'd1;
    end
end
```

## Integration Guidelines

### Core Integration Checklist

#### 1. Module Instantiation
```systemverilog
hps_io #(
    .CONF_STR(CONF_STR),      // Required: Configuration string
    .WIDE(0),                 // Optional: 0=8-bit, 1=16-bit file I/O
    .VDNUM(1),               // Optional: Number of virtual disks (1-10)
    .BLKSZ(2),               // Optional: Block size (0-7, default=2 for 512 bytes)
    .PS2DIV(1000),           // Optional: PS/2 clock divider (0=disabled)
    .PS2WE(0)                // Optional: PS/2 write enable
) hps_io (
    .clk_sys(clk_sys),       // System clock input
    .HPS_BUS(HPS_BUS),       // Connect to top-level HPS_BUS
    
    // Input device outputs (connect as needed)
    .joystick_0(joy0),
    .joystick_1(joy1),
    .ps2_key(ps2_key),
    .ps2_mouse(ps2_mouse),
    
    // Configuration outputs
    .status(status),
    .buttons(buttons),
    .forced_scandoubler(forced_scandoubler),
    
    // SD card interface (if needed)
    .sd_lba({sd_lba}),
    .sd_rd(sd_rd),
    .sd_wr(sd_wr),
    .sd_ack(sd_ack),
    .sd_buff_addr(sd_buff_addr),
    .sd_buff_dout(sd_buff_dout),
    .sd_buff_din(sd_buff_din),
    .sd_buff_wr(sd_buff_wr),
    .img_mounted(img_mounted),
    .img_size(img_size),
    .img_readonly(img_readonly),
    
    // File transfer interface (if needed)
    .ioctl_download(ioctl_download),
    .ioctl_index(ioctl_index),
    .ioctl_wr(ioctl_wr),
    .ioctl_addr(ioctl_addr),
    .ioctl_dout(ioctl_dout),
    .ioctl_wait(ioctl_wait),
    
    // RTC interface (if needed)
    .RTC(RTC),
    .TIMESTAMP(TIMESTAMP)
);
```

#### 2. Configuration String Development
```systemverilog
localparam CONF_STR = {
    "CORE_NAME;;",                                    // Core name and version
    "H0OOO,Aspect ratio,Original,Full Screen,[ARC1],[ARC2];", // Video options
    "H0-;",                                          // Separator
    "H0O[2],TV Mode,NTSC,PAL;",                      // TV standard
    "H0O[4:3],Scandoubler Fx,None,HQ2x,CRT 25%,CRT 50%;", // Video effects
    "H0-;",
    "H0O[6:5],Stereo mix,none,25%,50%,100%;",       // Audio options  
    "H0O[7],FM Sound,Off,On;",
    "H0-;",
    "H0T[8],Reset;",                                 // Reset option
    "H0R[0],Reset and close OSD;",                   // Reset and close
    "V,v",`BUILD_DATE                                // Version info
};
```

**Configuration String Syntax**:
- `H0`: Hide in direct video mode
- `O[bit]`: Single-bit option
- `O[end:start]`: Multi-bit option  
- `T[bit]`: Toggle button
- `R[bit]`: Reset button
- `-;`: Menu separator
- `V,v`: Version display

#### 3. Status Register Usage
```systemverilog
// Status bit assignments
wire [127:0] status;
wire tv_mode = status[2];
wire [1:0] scandoubler_fx = status[4:3];
wire [1:0] stereo_mix = status[6:5];
wire fm_sound = status[7];
wire reset_req = status[8];

// Status change detection
reg old_reset;
always @(posedge clk_sys) begin
    old_reset <= reset_req;
    if(~old_reset & reset_req) begin
        // Handle reset request
    end
end
```

#### 4. File Loading Integration
```systemverilog
// File download handling
always @(posedge clk_sys) begin
    if(ioctl_download && ioctl_wr) begin
        case(ioctl_index)
            8'h00: begin
                // ROM loading
                rom_addr <= ioctl_addr[rom_addr_width-1:0];
                rom_data <= ioctl_dout;
                rom_wr <= 1;
            end
            8'h01: begin
                // Secondary ROM/data loading
                ram_addr <= ioctl_addr[ram_addr_width-1:0];
                ram_data <= ioctl_dout;
                ram_wr <= 1;
            end
        endcase
    end else begin
        rom_wr <= 0;
        ram_wr <= 0;
    end
end

// Wait state generation for slow memory
assign ioctl_wait = rom_wr && rom_busy;
```

### Common Integration Patterns

#### 1. Input Device Integration
```systemverilog
// Joystick mapping
wire [31:0] joy0 = joystick_0;
wire [31:0] joy1 = joystick_1;

// Extract common controls
wire joy0_up    = joy0[3];
wire joy0_down  = joy0[2];  
wire joy0_left  = joy0[1];
wire joy0_right = joy0[0];
wire joy0_fire1 = joy0[4];
wire joy0_fire2 = joy0[5];

// Analog stick processing
wire signed [7:0] joy0_analog_x = joystick_l_analog_0[7:0];
wire signed [7:0] joy0_analog_y = joystick_l_analog_0[15:8];

// Combine digital and analog inputs
wire joy0_left_combined  = joy0_left  || (joy0_analog_x < -64);
wire joy0_right_combined = joy0_right || (joy0_analog_x > 64);
wire joy0_up_combined    = joy0_up    || (joy0_analog_y < -64);
wire joy0_down_combined  = joy0_down  || (joy0_analog_y > 64);
```

#### 2. PS/2 Integration
```systemverilog
// Keyboard interface
wire [10:0] ps2_key;
wire ps2_key_pressed = ps2_key[9];
wire ps2_key_extended = ps2_key[8];  
wire [7:0] ps2_key_code = ps2_key[7:0];
wire ps2_key_strobe = ps2_key[10] ^ ps2_key_strobe_r;

reg ps2_key_strobe_r;
always @(posedge clk_sys) begin
    ps2_key_strobe_r <= ps2_key[10];
    if(ps2_key_strobe) begin
        // Process keyboard input
        if(ps2_key_pressed) begin
            // Key press
            case(ps2_key_code)
                8'h1C: key_a_pressed <= 1;
                8'h32: key_b_pressed <= 1;
                // ... more keys
            endcase
        end else begin
            // Key release  
            case(ps2_key_code)
                8'h1C: key_a_pressed <= 0;
                8'h32: key_b_pressed <= 0;
                // ... more keys
            endcase
        end
    end
end

// Mouse interface
wire [24:0] ps2_mouse;
wire [15:0] ps2_mouse_ext;
wire mouse_strobe = ps2_mouse[24] ^ mouse_strobe_r;
wire [7:0] mouse_buttons = ps2_mouse[7:0];
wire signed [7:0] mouse_x = ps2_mouse[15:8];
wire signed [7:0] mouse_y = ps2_mouse[23:16]; 
wire signed [7:0] mouse_wheel = ps2_mouse_ext[7:0];

reg mouse_strobe_r;
always @(posedge clk_sys) begin
    mouse_strobe_r <= ps2_mouse[24];
    if(mouse_strobe) begin
        // Update mouse position
        mouse_pos_x <= mouse_pos_x + mouse_x;
        mouse_pos_y <= mouse_pos_y - mouse_y;  // Invert Y for screen coordinates
    end
end
```

#### 3. SD Card Integration  
```systemverilog
// SD card emulation for disk drives
parameter NUM_DISKS = 2;

wire [NUM_DISKS-1:0] img_mounted;
wire [31:0] sd_lba[NUM_DISKS];
wire [NUM_DISKS-1:0] sd_rd, sd_wr;
wire [NUM_DISKS-1:0] sd_ack;

// Disk controller integration
always @(posedge clk_sys) begin
    for(int i = 0; i < NUM_DISKS; i++) begin
        if(img_mounted[i]) begin
            // New disk image mounted
            disk_ready[i] <= 1;
            disk_sector_count[i] <= img_size >> 9;  // Convert bytes to 512-byte sectors
        end
        
        if(sd_rd[i] && !sd_ack[i]) begin
            // Start read operation
            disk_read_lba[i] <= sd_lba[i];
            disk_read_active[i] <= 1;
        end
        
        if(sd_wr[i] && !sd_ack[i]) begin
            // Start write operation  
            disk_write_lba[i] <= sd_lba[i];
            disk_write_active[i] <= 1;
        end
    end
end

// Buffer interface
always @(posedge clk_sys) begin
    if(sd_buff_wr) begin
        // Write data from ARM to buffer
        disk_buffer[sd_buff_addr] <= sd_buff_dout;
    end
    
    // Provide read data to ARM
    sd_buff_din[0] <= disk_buffer[sd_buff_addr];
    sd_buff_din[1] <= disk_buffer[sd_buff_addr];
end
```

#### 4. Video Integration
```systemverilog
// Video parameter monitoring
wire [15:0] video_param;
reg [4:0] video_param_num;

// Get video parameters from video_calc
assign video_param = vc_dout;

// Monitor for video mode changes
reg [7:0] old_vid_nres;
always @(posedge clk_sys) begin
    video_param_num <= 1;  // Read status parameter
    old_vid_nres <= video_param[7:0];
    
    if(old_vid_nres != video_param[7:0]) begin
        // Video mode changed, update core timing
        video_mode_changed <= 1;
    end
end
```

### Performance Considerations

#### 1. Clock Domain Crossing
```systemverilog
// Safe clock domain crossing for status
reg [127:0] status_sync;
reg [127:0] status_meta;

always @(posedge core_clk) begin
    status_meta <= status;      // Metastability stage
    status_sync <= status_meta; // Synchronized status
end
```

#### 2. FIFO Usage for High-Speed Data
```systemverilog
// FIFO for file loading
wire ioctl_fifo_full, ioctl_fifo_empty;
wire [15:0] ioctl_fifo_dout;

async_fifo #(.DATA_WIDTH(16), .ADDR_WIDTH(10)) ioctl_fifo (
    .wr_clk(clk_sys),
    .wr_data(ioctl_dout),
    .wr_en(ioctl_wr && !ioctl_fifo_full),
    .full(ioctl_fifo_full),
    
    .rd_clk(core_clk),  
    .rd_data(ioctl_fifo_dout),
    .rd_en(core_ready && !ioctl_fifo_empty),
    .empty(ioctl_fifo_empty)
);

assign ioctl_wait = ioctl_fifo_full;
```

#### 3. Memory Bandwidth Management
```systemverilog
// Arbitrate between ioctl and core memory access
wire mem_req_ioctl = ioctl_download && ioctl_wr;
wire mem_req_core = core_mem_req;

always @(posedge clk_sys) begin
    if(mem_req_ioctl && !mem_busy) begin
        // Prioritize file loading
        mem_addr <= ioctl_addr;
        mem_data <= ioctl_dout;
        mem_wr <= 1;
        mem_ack_ioctl <= 1;
    end else if(mem_req_core && !mem_busy && !mem_req_ioctl) begin
        // Core access when not loading
        mem_addr <= core_mem_addr;
        mem_data <= core_mem_data;
        mem_wr <= core_mem_wr;
        mem_ack_core <= 1;
    end else begin
        mem_wr <= 0;
        mem_ack_ioctl <= 0;
        mem_ack_core <= 0;
    end
end
```

## Debugging and Development

### Debug Signal Access

#### HPS_BUS Signal Monitoring
```systemverilog
// Monitor HPS_BUS signals for debugging
wire debug_io_strobe = HPS_BUS[33];
wire debug_io_enable = HPS_BUS[34];  
wire debug_fp_enable = HPS_BUS[35];
wire [15:0] debug_io_din = HPS_BUS[31:16];
wire [15:0] debug_io_dout = HPS_BUS[15:0];

// Debug state machine
reg [3:0] debug_state;
always @(posedge clk_sys) begin
    case(debug_state)
        0: if(debug_io_strobe) debug_state <= 1;
        1: if(debug_io_enable) debug_state <= 2;
        2: if(~debug_io_enable) debug_state <= 0;
    endcase
end
```

#### Command Logging
```systemverilog
// Command trace buffer
reg [15:0] cmd_trace[255:0];
reg [7:0] cmd_trace_ptr;

always @(posedge clk_sys) begin
    if(io_strobe && byte_cnt == 0) begin
        cmd_trace[cmd_trace_ptr] <= io_din;
        cmd_trace_ptr <= cmd_trace_ptr + 1;
    end
end
```

### Common Debug Scenarios

#### 1. File Loading Issues
```systemverilog
// File loading debug
reg [31:0] ioctl_debug_count;
reg [15:0] ioctl_debug_last_data;
reg [26:0] ioctl_debug_last_addr;

always @(posedge clk_sys) begin
    if(ioctl_download && ioctl_wr) begin
        ioctl_debug_count <= ioctl_debug_count + 1;
        ioctl_debug_last_data <= ioctl_dout;
        ioctl_debug_last_addr <= ioctl_addr;
    end
    
    if(!ioctl_download) begin
        ioctl_debug_count <= 0;
    end
end

// LED indicators for debug
assign LED_USER = ioctl_download;
assign LED_DISK[0] = |ioctl_debug_count[15:0];
assign LED_POWER[0] = ioctl_wait;
```

#### 2. Input Device Debug
```systemverilog
// Joystick input debug
reg [31:0] joy_debug_count[6];
reg [31:0] joy_debug_last[6];

always @(posedge clk_sys) begin
    if(joystick_0 != joy_debug_last[0]) begin
        joy_debug_count[0] <= joy_debug_count[0] + 1;
        joy_debug_last[0] <= joystick_0;
    end
    // ... repeat for other joysticks
end

// PS/2 debug
reg [10:0] ps2_debug_last_key;
reg [31:0] ps2_debug_key_count;

always @(posedge clk_sys) begin
    if(ps2_key != ps2_debug_last_key) begin
        ps2_debug_key_count <= ps2_debug_key_count + 1;
        ps2_debug_last_key <= ps2_key;
    end
end
```

#### 3. Configuration Debug
```systemverilog
// Status register debug
reg [127:0] status_debug_last;
reg [31:0] status_debug_change_count;

always @(posedge clk_sys) begin
    if(status != status_debug_last) begin
        status_debug_change_count <= status_debug_change_count + 1;
        status_debug_last <= status;
    end
end

// Configuration string access debug
reg [31:0] conf_str_access_count;
reg [7:0] conf_str_last_byte;

always @(posedge clk_sys) begin
    if(cmd == 'h14 && io_strobe && byte_cnt > 0) begin
        conf_str_access_count <= conf_str_access_count + 1;
        conf_str_last_byte <= conf_byte;
    end
end
```

### Test Patterns and Validation

#### 1. Loopback Tests
```systemverilog
// Joystick rumble loopback test
reg [15:0] rumble_test_pattern;
reg [3:0] rumble_test_counter;

always @(posedge clk_sys) begin
    rumble_test_counter <= rumble_test_counter + 1;
    if(&rumble_test_counter) begin
        rumble_test_pattern <= rumble_test_pattern + 1;
    end
end

assign joystick_0_rumble = status[15] ? rumble_test_pattern : 16'h0000;
```

#### 2. Communication Integrity Tests
```systemverilog
// Command echo test
reg [15:0] echo_test_data;
reg echo_test_enable;

always @(posedge clk_sys) begin
    if(cmd == 'h3F && byte_cnt == 1) begin  // Custom test command
        echo_test_data <= io_din;
        echo_test_enable <= 1;
    end
    
    if(cmd == 'h3F && byte_cnt == 2) begin
        io_dout <= echo_test_enable ? echo_test_data : 16'hDEAD;
    end
end
```

#### 3. Timing Validation
```systemverilog
// Video timing validation
reg [31:0] vsync_period_measure;
reg [31:0] hsync_period_measure;
reg vsync_last, hsync_last;

always @(posedge clk_sys) begin
    vsync_last <= vs;
    hsync_last <= hs;
    
    vsync_period_measure <= vsync_period_measure + 1;
    hsync_period_measure <= hsync_period_measure + 1;
    
    if(vsync_last && !vs) vsync_period_measure <= 0;
    if(hsync_last && !hs) hsync_period_measure <= 0;
end
```

### ARM-Side Debug Support

#### 1. Debug Output Functions
```c
// ARM side debug helpers
void debug_print_hps_io_status(void) {
    printf("=== HPS_IO Debug Status ===\n");
    printf("IO Enable: %d\n", (fpga_gpi_read() >> 20) & 1);
    printf("FPGA Enable: %d\n", (fpga_gpi_read() >> 18) & 1);
    printf("OSD Enable: %d\n", (fpga_gpi_read() >> 19) & 1);
}

void debug_test_communication(void) {
    uint16_t test_data = 0xA5A5;
    uint16_t result;
    
    // Test echo command
    spi_uio_cmd_cont(0x3F);
    spi_uio_cmd_cont(test_data);
    result = spi_uio_cmd(0);
    
    printf("Echo test: sent 0x%04X, received 0x%04X %s\n", 
           test_data, result, (result == test_data) ? "PASS" : "FAIL");
}
```

#### 2. Configuration Validation
```c
void debug_validate_config_string(void) {
    char config_str[1024];
    int len = 0;
    
    spi_uio_cmd_cont(UIO_GET_STRING);
    while(len < sizeof(config_str)-1) {
        config_str[len] = spi_uio_cmd_cont(0);
        if(config_str[len] == 0) break;
        len++;
    }
    spi_uio_cmd(0);
    
    printf("Config string (%d bytes): %s\n", len, config_str);
}
```

This comprehensive documentation provides developers with the complete technical understanding needed to effectively integrate and utilize the hps_io module in MiSTer cores, from basic functionality to advanced debugging and optimization techniques.

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"id": "1", "content": "Search for hps_io module files in codebase", "status": "completed", "priority": "high"}, {"id": "2", "content": "Analyze hps_io interface and signals", "status": "completed", "priority": "high"}, {"id": "3", "content": "Document ARM-FPGA communication protocol", "status": "completed", "priority": "high"}, {"id": "4", "content": "Examine configuration and data transfer mechanisms", "status": "completed", "priority": "high"}, {"id": "5", "content": "Document I/O interfaces and peripherals", "status": "completed", "priority": "medium"}, {"id": "6", "content": "Create comprehensive hps_io documentation", "status": "completed", "priority": "high"}]