# MiSTer HDMI System - Comprehensive Technical Documentation

## Table of Contents

1. [Overview](#overview)
2. [Hardware Architecture](#hardware-architecture)
3. [HDMI PLL and Clock Management](#hdmi-pll-and-clock-management)
4. [Video Processing Pipeline](#video-processing-pipeline)
5. [Audio Integration](#audio-integration)
6. [ARM-Side Control System](#arm-side-control-system)
7. [EDID and Display Detection](#edid-and-display-detection)
8. [Video Standards and Compliance](#video-standards-and-compliance)
9. [Advanced Features](#advanced-features)
10. [Integration and Synchronization](#integration-and-synchronization)
11. [Performance and Optimization](#performance-and-optimization)
12. [Development and Debugging](#development-and-debugging)

## Overview

The MiSTer HDMI system represents a sophisticated digital video output implementation designed for pixel-perfect reproduction of retro gaming and computing systems. The architecture combines advanced FPGA-based video processing with ARM-side control software to provide professional-grade video output with extensive customization capabilities.

### Key Features

- **Pixel-Perfect Video Output**: Preserves original video timing and quality
- **Dynamic PLL Reconfiguration**: Real-time pixel clock adjustment for VSync matching
- **Advanced Scaling Engine**: Multiple interpolation algorithms with custom filtering
- **Professional Audio Integration**: I2S and S/PDIF digital audio output
- **Variable Refresh Rate**: FreeSync/AdaptiveSync compatibility
- **EDID-Based Mode Detection**: Automatic display capability detection
- **Low-Latency Operation**: Direct video mode for minimal input lag
- **Extensive Compatibility**: Support for modern displays and legacy timing

## Hardware Architecture

### Physical Interface Implementation

The MiSTer HDMI implementation utilizes a comprehensive set of dedicated pins on the DE10-Nano FPGA board:

#### Video Signal Pins
```verilog
// Primary video outputs
output        HDMI_TX_CLK,     // Pixel clock (DDR output)
output        HDMI_TX_DE,      // Data enable signal
output [23:0] HDMI_TX_D,       // 24-bit RGB data (8 bits per component)
output        HDMI_TX_HS,      // Horizontal sync
output        HDMI_TX_VS,      // Vertical sync
input         HDMI_TX_INT,     // Interrupt from HDMI encoder
```

#### Control Interface Pins
```verilog
// I2C interface for EDID reading and encoder control
output        HDMI_I2C_SCL,    // I2C clock line
inout         HDMI_I2C_SDA,    // I2C data line (bidirectional)
```

#### Audio Interface Pins
```verilog
// Digital audio output via I2S
output        HDMI_MCLK,       // Master clock (audio reference)
output        HDMI_SCLK,       // Serial clock (bit clock)
output        HDMI_LRCLK,      // Left/Right clock (frame sync)
output        HDMI_I2S,        // I2S data stream
```

### DDR Clock Generation

The HDMI pixel clock uses a sophisticated Double Data Rate (DDR) output mechanism to achieve the required frequencies:

```verilog
// DDR output for HDMI clock generation
altddio_out #(
    .extend_oe_disable("OFF"),
    .intended_device_family("Cyclone V"),
    .invert_output("OFF"),
    .lpm_hint("UNUSED"),
    .lpm_type("altddio_out"),
    .oe_reg("UNREGISTERED"),
    .power_up_high("OFF"),
    .width(1)
) hdmi_clk_ddr (
    .datain_h(1'b0),           // High phase data
    .datain_l(1'b1),           // Low phase data  
    .outclock(hdmi_tx_clk),    // Source clock from PLL
    .dataout(HDMI_TX_CLK)      // DDR output to HDMI encoder
);
```

This configuration effectively doubles the output frequency by toggling the output on both rising and falling edges of the internal clock, enabling generation of high-frequency pixel clocks from lower-frequency PLL outputs.

### Pin Assignment and Constraints

```tcl
# HDMI pin assignments for DE10-Nano
set_location_assignment PIN_AG5  -to HDMI_TX_CLK
set_location_assignment PIN_AD12 -to HDMI_TX_DE  
set_location_assignment PIN_AE12 -to HDMI_TX_D[23]
set_location_assignment PIN_W8   -to HDMI_TX_D[22]
# ... (continued for all data pins)
set_location_assignment PIN_U10  -to HDMI_TX_HS
set_location_assignment PIN_Y13  -to HDMI_TX_VS
set_location_assignment PIN_T13  -to HDMI_TX_INT

# I2C interface pins
set_location_assignment PIN_U9   -to HDMI_I2C_SCL
set_location_assignment PIN_AG4  -to HDMI_I2C_SDA

# I/O standards for signal integrity
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to HDMI_TX_*
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to HDMI_I2C_*
```

## HDMI PLL and Clock Management

### Primary HDMI PLL Architecture

The HDMI PLL system consists of multiple interconnected components providing flexible clock generation:

#### Base PLL Module (`pll_hdmi.v`)
```verilog
module pll_hdmi (
    input  wire        refclk,            // 50 MHz reference clock
    input  wire        rst,               // Reset signal
    output wire        outclk_0,          // Generated pixel clock
    input  wire [63:0] reconfig_to_pll,   // Reconfiguration interface
    output wire [63:0] reconfig_from_pll  // Status interface
);
```

**Key Specifications**:
- **Reference Clock**: 50 MHz crystal oscillator
- **Output Range**: 25 MHz to 300 MHz (typical video frequencies)
- **Resolution**: Fractional-N synthesis with high precision
- **Reconfiguration**: Dynamic frequency changes without reset

#### Advanced PLL Adjustment System (`pll_hdmi_adj.vhd`)

This sophisticated VHDL module provides real-time pixel clock adjustment for optimal video synchronization:

```vhdl
entity pll_hdmi_adj is
    generic (
        FRAC_BITS : natural := 32    -- Fractional precision
    );
    port (
        clk_sys        : in  std_logic;
        reset_n        : in  std_logic;
        
        -- Timing measurement inputs
        vs_in          : in  std_logic;    -- Input vertical sync
        vs_out         : in  std_logic;    -- Output vertical sync
        
        -- PLL control outputs
        pll_reconfig   : out std_logic_vector(63 downto 0);
        pll_busy       : in  std_logic;
        
        -- Configuration
        adj_enable     : in  std_logic;    -- Enable adjustment
        target_freq    : in  std_logic_vector(31 downto 0);
        
        -- Status outputs
        locked         : out std_logic;
        freq_delta     : out signed(15 downto 0)
    );
end entity;
```

**Adjustment Algorithm**:
1. **Phase Detection**: Measures phase relationship between input and output VSync
2. **Frequency Calculation**: Determines required pixel clock adjustment
3. **PID Control**: Implements proportional-integral-derivative control loop
4. **Dynamic Update**: Applies changes via PLL reconfiguration interface

#### PLL Reconfiguration Interface (`pll_cfg_hdmi.v`)

The reconfiguration system enables dynamic frequency changes:

```verilog
// PLL reconfiguration parameters
typedef struct {
    logic [7:0]  m_counter;      // Integer multiplier
    logic [31:0] m_frac;         // Fractional multiplier  
    logic [7:0]  c0_counter;     // Output divider
    logic [2:0]  bandwidth;      // Loop bandwidth setting
    logic [7:0]  charge_pump;    // Charge pump current
} pll_config_t;

// Reconfiguration state machine
typedef enum {
    IDLE,
    WAIT_LOCK,
    UPDATE_M,
    UPDATE_C,
    UPDATE_FRAC,
    COMPLETE
} reconfig_state_t;
```

### Clock Distribution Network

```verilog
// Clock distribution for HDMI subsystem
wire clk_hdmi_pixel;    // Primary pixel clock
wire clk_hdmi_tmds;     // TMDS serializer clock (10x pixel)
wire clk_audio_mclk;    // Audio master clock
wire clk_audio_sclk;    // Audio serial clock

// Clock enables for different components
wire ce_pixel;          // Pixel rate clock enable
wire ce_audio;          // Audio sample rate clock enable
```

### Frequency Calculation and Synthesis

The system calculates precise PLL parameters for any target frequency:

```cpp
// ARM-side PLL calculation
struct pll_calc_t {
    uint32_t m_int;        // Integer multiplier
    uint32_t m_frac;       // Fractional multiplier (32-bit)
    uint32_t c0_div;       // Output divider
    uint32_t actual_freq;  // Resulting frequency
    uint32_t error_ppm;    // Error in parts-per-million
};

pll_calc_t calculate_pll_params(uint32_t target_freq) {
    pll_calc_t result;
    
    // Target: 50 MHz * (M + M_FRAC/2^32) / C0 = target_freq
    // Optimize for minimum error and acceptable dividers
    
    uint32_t best_error = UINT32_MAX;
    
    for (uint32_t c0 = 1; c0 <= 512; c0++) {
        uint64_t m_total = (uint64_t)target_freq * c0;
        uint32_t m_int = m_total / 50000000;
        uint64_t m_frac_64 = ((m_total % 50000000) << 32) / 50000000;
        uint32_t m_frac = (uint32_t)m_frac_64;
        
        // Calculate actual frequency
        uint64_t actual = (50000000ULL * (((uint64_t)m_int << 32) + m_frac)) >> 32;
        actual /= c0;
        
        uint32_t error = abs((int64_t)actual - target_freq);
        
        if (error < best_error && m_int >= 1 && m_int <= 255) {
            best_error = error;
            result.m_int = m_int;
            result.m_frac = m_frac;
            result.c0_div = c0;
            result.actual_freq = actual;
            result.error_ppm = (error * 1000000) / target_freq;
        }
    }
    
    return result;
}
```

## Video Processing Pipeline

### Video Data Flow Architecture

The MiSTer video processing pipeline consists of multiple specialized stages:

```
Core Video → Video Mixer → Scanlines → Scaler → OSD → Shadow Mask → HDMI Output
     ↓            ↓           ↓          ↓       ↓        ↓         ↓
   Raw RGB    Border/Mix   CRT Effect  Scale   Menu    CRT Sim   Encoder
```

### Video Mixer Implementation (`video_mixer.sv`)

The video mixer handles multiple video sources and effects:

```systemverilog
module video_mixer #(
    parameter LINE_LENGTH = 2048,
    parameter HALF_DEPTH  = 0,
    parameter GAMMA       = 0
) (
    // Input video
    input             clk_vid,
    input             ce_pix,
    input [7:0]       R,
    input [7:0]       G, 
    input [7:0]       B,
    input             HSync,
    input             VSync,
    input             HBlank,
    input             VBlank,
    
    // Output video
    output reg [7:0]  VGA_R,
    output reg [7:0]  VGA_G,
    output reg [7:0]  VGA_B,
    output reg        VGA_HS,
    output reg        VGA_VS,
    output reg        VGA_DE,
    
    // Control signals
    input             scandoubler_disable,
    input             hq2x_enable,
    input [1:0]       scanlines,
    input [7:0]       gamma_bus
);
```

**Key Features**:
- **Scandoubler**: Line doubling for VGA output compatibility
- **HQ2x Scaling**: High-quality 2x scaling algorithm for pixel art
- **Scanline Generation**: Configurable scanline effects (25%, 50%, 75%)
- **Gamma Correction**: Programmable gamma curve application
- **Border Generation**: Configurable border colors and patterns

### Advanced Scaler Integration (ASCAL)

The ASCAL (Advanced Scaler) provides sophisticated video scaling:

```systemverilog
ascal #(
    .RAMBASE(32'h20000000),    // DDR3 base address
    .RAMSIZE(32'h00800000),    // 8MB framebuffer
    .INTER(1),                 // Interlace detection
    .HEADER(1),                // Include frame headers
    .DOWNSCALE(1),             // Downscaling support
    .ADAPTIVE(1),              // Adaptive filtering
    .FRAC(8),                  // 8-bit fractional precision
    .OHRES(2304),              // Max output width
    .N_DW(128)                 // 128-bit memory interface
) ascal_inst (
    // Input from video mixer
    .i_clk(clk_vid),
    .i_r(mixer_r), .i_g(mixer_g), .i_b(mixer_b),
    .i_hs(mixer_hs), .i_vs(mixer_vs), .i_de(mixer_de),
    
    // Output to HDMI
    .o_clk(clk_hdmi),
    .o_r(hdmi_r), .o_g(hdmi_g), .o_b(hdmi_b),
    .o_hs(hdmi_hs), .o_vs(hdmi_vs), .o_de(hdmi_de),
    
    // Memory interface
    .avl_clk(clk_mem),
    .avl_address(mem_addr),
    .avl_writedata(mem_wdata),
    .avl_readdata(mem_rdata),
    
    // Configuration
    .mode(scaler_mode),
    .format(pixel_format),
    .run(scaler_enable)
);
```

### Video Timing Generation

The system generates precise video timing for any resolution:

```systemverilog
// Video timing generator
module video_timing_gen (
    input  wire        clk,
    input  wire        reset,
    
    // Timing parameters
    input  wire [11:0] h_total,
    input  wire [11:0] h_sync_start,
    input  wire [11:0] h_sync_end,
    input  wire [11:0] h_active,
    input  wire [11:0] v_total,
    input  wire [11:0] v_sync_start,
    input  wire [11:0] v_sync_end,
    input  wire [11:0] v_active,
    
    // Generated signals
    output reg         h_sync,
    output reg         v_sync,
    output reg         data_enable,
    output reg [11:0]  h_count,
    output reg [11:0]  v_count
);

always @(posedge clk) begin
    if (reset) begin
        h_count <= 0;
        v_count <= 0;
    end else begin
        // Horizontal counter
        if (h_count == h_total - 1) begin
            h_count <= 0;
            
            // Vertical counter
            if (v_count == v_total - 1)
                v_count <= 0;
            else
                v_count <= v_count + 1;
        end else begin
            h_count <= h_count + 1;
        end
        
        // Generate sync signals
        h_sync <= (h_count >= h_sync_start) && (h_count < h_sync_end);
        v_sync <= (v_count >= v_sync_start) && (v_count < v_sync_end);
        
        // Generate data enable
        data_enable <= (h_count < h_active) && (v_count < v_active);
    end
end
endmodule
```

## Audio Integration

### I2S Digital Audio Implementation

The I2S (Inter-IC Sound) interface provides professional digital audio output:

```systemverilog
module i2s_transmitter #(
    parameter DATA_WIDTH = 16
) (
    input  wire                    clk,          // System clock
    input  wire                    reset,
    
    // Audio data input
    input  wire [DATA_WIDTH-1:0]   left_data,
    input  wire [DATA_WIDTH-1:0]   right_data,
    input  wire                    data_valid,
    
    // I2S output signals
    output reg                     mclk,         // Master clock
    output reg                     sclk,         // Serial clock (bit clock)
    output reg                     lrclk,        // Left/Right clock (frame sync)
    output reg                     sdata,        // Serial data
    
    // Configuration
    input  wire [7:0]              sample_rate,  // Sample rate selection
    input  wire                    enable
);

// Clock divider for sample rate generation
reg [15:0] mclk_div;
reg [7:0]  sclk_div;
reg [5:0]  lrclk_div;

// Sample rate configuration
always @(*) begin
    case (sample_rate)
        8'd48:  mclk_div = 1024;  // 48 kHz: 49.152 MHz / 1024 = 48 kHz
        8'd96:  mclk_div = 512;   // 96 kHz: 49.152 MHz / 512 = 96 kHz
        default: mclk_div = 1024;
    endcase
end

// I2S transmission logic
reg [4:0] bit_counter;
reg [31:0] shift_reg;
reg frame_sync;

always @(posedge clk) begin
    if (reset) begin
        bit_counter <= 0;
        shift_reg <= 0;
        lrclk <= 0;
        sdata <= 0;
    end else if (enable) begin
        // Generate bit clock
        if (sclk_div == 0) begin
            sclk <= ~sclk;
            sclk_div <= 31;  // 64x oversampling
            
            if (sclk) begin  // Rising edge of bit clock
                // Load new frame data
                if (bit_counter == 0) begin
                    if (lrclk) begin
                        shift_reg <= {right_data, 16'h0000};  // Right channel
                    end else begin
                        shift_reg <= {left_data, 16'h0000};   // Left channel
                    end
                end
                
                // Shift out data
                sdata <= shift_reg[31];
                shift_reg <= {shift_reg[30:0], 1'b0};
                
                // Update counters
                bit_counter <= bit_counter + 1;
                if (bit_counter == 31) begin
                    bit_counter <= 0;
                    lrclk <= ~lrclk;  // Toggle L/R clock
                end
            end
        end else begin
            sclk_div <= sclk_div - 1;
        end
    end
end
endmodule
```

### S/PDIF Digital Audio Implementation

S/PDIF provides another professional digital audio output option:

```systemverilog
module spdif_transmitter (
    input  wire        clk,
    input  wire        reset,
    
    // Audio input
    input  wire [15:0] left_sample,
    input  wire [15:0] right_sample,
    input  wire        sample_valid,
    
    // S/PDIF output
    output reg         spdif_out,
    
    // Configuration
    input  wire [7:0]  sample_rate
);

// Biphase Mark Code (BMC) encoder
reg [31:0] frame_data;
reg [5:0]  bit_counter;
reg        parity;
reg        channel;  // 0=left, 1=right

// Frame structure for S/PDIF
always @(*) begin
    case (channel)
        1'b0: begin  // Left channel (Channel A)
            frame_data = {
                4'b0001,           // Preamble Z (start of frame)
                left_sample,       // Audio data
                4'b0000,           // Auxiliary data
                1'b0,              // Copyright bit
                3'b000,            // Category code
                parity,            // Parity bit
                1'b0,              // Channel status
                1'b0,              // User data
                1'b1               // Validity bit
            };
        end
        1'b1: begin  // Right channel (Channel B)
            frame_data = {
                4'b0010,           // Preamble Y (right channel)
                right_sample,      // Audio data
                4'b0000,           // Auxiliary data
                1'b0,              // Copyright bit
                3'b000,            // Category code
                parity,            // Parity bit
                1'b0,              // Channel status
                1'b0,              // User data
                1'b1               // Validity bit
            };
        end
    endcase
end

// BMC encoding and transmission
reg bmc_clock;
reg last_bit;

always @(posedge clk) begin
    if (reset) begin
        bit_counter <= 0;
        channel <= 0;
        bmc_clock <= 0;
        last_bit <= 0;
        spdif_out <= 0;
    end else begin
        // BMC encoding: 
        // '1' = transition at beginning + transition at middle
        // '0' = transition at beginning only
        
        bmc_clock <= ~bmc_clock;
        
        if (bmc_clock) begin  // First half of bit period
            spdif_out <= ~last_bit;  // Always transition at start
        end else begin  // Second half of bit period
            if (frame_data[31 - bit_counter]) begin
                spdif_out <= ~spdif_out;  // Additional transition for '1'
            end
            
            last_bit <= spdif_out;
            bit_counter <= bit_counter + 1;
            
            if (bit_counter == 31) begin
                bit_counter <= 0;
                channel <= ~channel;
            end
        end
    end
end
endmodule
```

### Audio Clock Generation

Dedicated audio PLLs provide jitter-free clocks:

```systemverilog
// Audio PLL for precise sample rates
pll_audio audio_pll_inst (
    .refclk(clk_50m),
    .rst(reset),
    
    // Multiple audio clocks
    .outclk_0(clk_audio_49152),  // 49.152 MHz (for 48/96 kHz)
    .outclk_1(clk_audio_45158),  // 45.158 MHz (for 44.1/88.2 kHz)
    .outclk_2(clk_audio_12288),  // 12.288 MHz (256x 48 kHz)
    
    .locked(audio_pll_locked)
);
```

## ARM-Side Control System

### Video Configuration Management (`video.cpp`)

The ARM processor manages comprehensive video configuration:

```cpp
// Video mode structure
typedef struct {
    char name[32];
    uint16_t width, height;
    uint16_t h_total, h_sync_start, h_sync_end;
    uint16_t v_total, v_sync_start, v_sync_end;
    uint32_t pixel_clock;    // Hz
    uint8_t progressive;     // 0=interlaced, 1=progressive
    uint8_t h_pol, v_pol;    // Sync polarities
} video_mode_t;

// Standard video modes
static const video_mode_t video_modes[] = {
    {"720p60",   1280, 720,  1650, 1390, 1430, 750, 725, 730, 74250000, 1, 1, 1},
    {"1080p60",  1920, 1080, 2200, 2008, 2052, 1125, 1084, 1089, 148500000, 1, 1, 1},
    {"1440p60",  2560, 1440, 2720, 2608, 2652, 1481, 1444, 1449, 241500000, 1, 1, 1},
    {"4K30",     3840, 2160, 4400, 4016, 4104, 2250, 2168, 2178, 297000000, 1, 1, 1},
    {"VGA60",    640,  480,  800,  656,  752,  525,  490,  492,  25175000, 1, 0, 0},
};

// Video mode setting function
void set_video_mode(int mode_index) {
    const video_mode_t *mode = &video_modes[mode_index];
    
    // Calculate PLL parameters
    pll_calc_t pll = calculate_pll_params(mode->pixel_clock);
    
    // Configure HDMI PLL
    fpga_core_write(PLL_RECONFIG_BASE + 0x00, pll.m_int);
    fpga_core_write(PLL_RECONFIG_BASE + 0x04, pll.m_frac);
    fpga_core_write(PLL_RECONFIG_BASE + 0x08, pll.c0_div);
    
    // Set video timing parameters
    fpga_core_write(VIDEO_TIMING_BASE + 0x00, 
        (mode->h_total << 16) | mode->h_sync_start);
    fpga_core_write(VIDEO_TIMING_BASE + 0x04,
        (mode->h_sync_end << 16) | mode->width);
    fpga_core_write(VIDEO_TIMING_BASE + 0x08,
        (mode->v_total << 16) | mode->v_sync_start);
    fpga_core_write(VIDEO_TIMING_BASE + 0x0C,
        (mode->v_sync_end << 16) | mode->height);
    
    // Apply configuration
    fpga_core_write(VIDEO_CONTROL_BASE, 0x01);  // Enable new mode
    
    printf("Set video mode: %s (%dx%d@%.2fHz)\n",
           mode->name, mode->width, mode->height,
           (float)mode->pixel_clock / (mode->h_total * mode->v_total));
}
```

### Dynamic VSync Adjustment

VSync adjustment provides smooth video reproduction:

```cpp
// VSync adjustment state
typedef struct {
    uint32_t target_freq;      // Target pixel clock
    uint32_t current_freq;     // Current pixel clock
    int32_t  freq_delta;       // Frequency adjustment
    uint8_t  adjustment_active; // Currently adjusting
    uint8_t  lock_status;      // PLL lock status
} vsync_adjust_t;

static vsync_adjust_t vsync_state = {0};

void update_vsync_adjustment() {
    if (!cfg.vsync_adjust) return;
    
    // Read timing measurements from FPGA
    uint32_t input_period = fpga_core_read(TIMING_MEASURE_BASE + 0x00);
    uint32_t output_period = fpga_core_read(TIMING_MEASURE_BASE + 0x04);
    
    // Calculate frequency difference
    int32_t period_delta = input_period - output_period;
    
    if (abs(period_delta) > VSYNC_TOLERANCE) {
        // Calculate new pixel clock frequency
        uint64_t new_freq = (uint64_t)vsync_state.current_freq * input_period;
        new_freq /= output_period;
        
        // Limit adjustment range
        if (new_freq > vsync_state.target_freq * 1.05f ||
            new_freq < vsync_state.target_freq * 0.95f) {
            new_freq = vsync_state.target_freq;
        }
        
        // Calculate new PLL parameters
        pll_calc_t pll = calculate_pll_params(new_freq);
        
        if (pll.error_ppm < 100) {  // Accept if error < 100 ppm
            // Apply adjustment
            fpga_core_write(PLL_RECONFIG_BASE + 0x04, pll.m_frac);
            fpga_core_write(PLL_RECONFIG_BASE + 0x10, 0x01);  // Trigger update
            
            vsync_state.current_freq = new_freq;
            vsync_state.adjustment_active = 1;
        }
    }
}
```

### Configuration System Integration (`cfg.cpp`)

HDMI settings are integrated into the comprehensive configuration system:

```cpp
// HDMI-related configuration parameters
typedef struct {
    // Video settings
    uint8_t  hdmi_limited;        // Color range (0=full, 1=limited)
    uint8_t  dvi_mode;           // DVI mode (no audio)
    uint8_t  hdmi_audio_96k;     // High sample rate audio
    uint8_t  direct_video;       // Bypass scaler
    uint8_t  vsync_adjust;       // Dynamic refresh adjustment
    float    refresh_min;        // Minimum refresh rate
    float    refresh_max;        // Maximum refresh rate
    uint8_t  video_info;         // Show video info overlay
    
    // Advanced features
    uint8_t  vrr_mode;           // Variable refresh rate
    uint8_t  vrr_min_framerate;  // VRR minimum
    uint8_t  vrr_max_framerate;  // VRR maximum
    uint8_t  hdmi_game_mode;     // Gaming mode optimizations
    
    // Color adjustment
    uint8_t  video_brightness;   // Brightness adjustment
    uint8_t  video_contrast;     // Contrast adjustment
    uint8_t  video_saturation;   // Saturation adjustment
    uint16_t video_hue;          // Hue adjustment
    
    // HDR support
    uint8_t  hdr;                // HDR enable
    uint16_t hdr_max_nits;       // Peak brightness
    uint16_t hdr_avg_nits;       // Average brightness
} hdmi_config_t;

// Configuration parsing
void parse_hdmi_config(const char *line) {
    if (strstr(line, "hdmi_limited=")) {
        cfg.hdmi_limited = atoi(strchr(line, '=') + 1);
    }
    else if (strstr(line, "vsync_adjust=")) {
        cfg.vsync_adjust = atoi(strchr(line, '=') + 1);
    }
    else if (strstr(line, "video_mode=")) {
        // Parse custom video mode: width,height,refresh
        char *token = strtok(strchr(line, '=') + 1, ",");
        if (token) custom_mode.width = atoi(token);
        token = strtok(NULL, ",");
        if (token) custom_mode.height = atoi(token);
        token = strtok(NULL, ",");
        if (token) custom_mode.refresh = atof(token);
    }
    // ... additional parsing
}
```

## EDID and Display Detection

### EDID Reading and Parsing

The system implements comprehensive EDID (Extended Display Identification Data) support:

```cpp
// EDID structure definitions
typedef struct {
    uint8_t header[8];           // EDID header signature
    uint16_t manufacturer_id;    // Manufacturer ID
    uint16_t product_code;       // Product code
    uint32_t serial_number;      // Serial number
    uint8_t week;                // Week of manufacture
    uint8_t year;                // Year of manufacture
    uint8_t edid_version;        // EDID version
    uint8_t edid_revision;       // EDID revision
} edid_header_t;

typedef struct {
    uint16_t pixel_clock;        // Pixel clock in 10 kHz units
    uint16_t h_active;           // Horizontal active pixels
    uint16_t h_blanking;         // Horizontal blanking
    uint16_t v_active;           // Vertical active lines
    uint16_t v_blanking;         // Vertical blanking
    uint16_t h_sync_offset;      // Horizontal sync offset
    uint16_t h_sync_width;       // Horizontal sync width
    uint8_t  v_sync_offset;      // Vertical sync offset
    uint8_t  v_sync_width;       // Vertical sync width
    uint8_t  h_image_size;       // Horizontal image size (mm)
    uint8_t  v_image_size;       // Vertical image size (mm)
    uint8_t  flags;              // Various flags
} detailed_timing_t;

// EDID reading function
static int read_edid(uint8_t *edid_data) {
    int fd = open("/dev/i2c-1", O_RDWR);
    if (fd < 0) return -1;
    
    // Set I2C slave address (0x50 for EDID)
    if (ioctl(fd, I2C_SLAVE, 0x50) < 0) {
        close(fd);
        return -1;
    }
    
    // Read 256 bytes of EDID data
    for (int i = 0; i < 256; i++) {
        int result = i2c_smbus_read_byte_data(fd, i);
        if (result < 0) {
            close(fd);
            return -1;
        }
        edid_data[i] = result & 0xFF;
    }
    
    close(fd);
    
    // Validate EDID checksum
    uint8_t checksum = 0;
    for (int i = 0; i < 128; i++) {
        checksum += edid_data[i];
    }
    
    return (checksum == 0) ? 0 : -1;
}

// EDID parsing function
video_mode_t parse_preferred_timing(uint8_t *edid) {
    video_mode_t mode = {0};
    
    // Preferred timing is in bytes 54-71 of base EDID
    detailed_timing_t *timing = (detailed_timing_t *)(edid + 54);
    
    if (timing->pixel_clock > 0) {
        uint32_t pixel_clk = timing->pixel_clock * 10000;  // Convert to Hz
        
        mode.width = timing->h_active;
        mode.height = timing->v_active;
        mode.pixel_clock = pixel_clk;
        
        // Calculate total timings
        mode.h_total = timing->h_active + timing->h_blanking;
        mode.v_total = timing->v_active + timing->v_blanking;
        
        // Calculate sync timings
        mode.h_sync_start = timing->h_active + timing->h_sync_offset;
        mode.h_sync_end = mode.h_sync_start + timing->h_sync_width;
        mode.v_sync_start = timing->v_active + timing->v_sync_offset;
        mode.v_sync_end = mode.v_sync_start + timing->v_sync_width;
        
        // Extract sync polarities
        mode.h_pol = (timing->flags & 0x02) ? 1 : 0;
        mode.v_pol = (timing->flags & 0x04) ? 1 : 0;
        
        // Calculate refresh rate
        float refresh = (float)pixel_clk / (mode.h_total * mode.v_total);
        
        snprintf(mode.name, sizeof(mode.name), "%dx%d@%.1f", 
                mode.width, mode.height, refresh);
    }
    
    return mode;
}
```

### Display Capability Detection

Advanced display feature detection:

```cpp
// Display capability structure
typedef struct {
    uint8_t supports_hdmi;       // HDMI vs DVI
    uint8_t supports_audio;      // Audio capability
    uint8_t supports_ycbcr444;   // YCbCr 4:4:4
    uint8_t supports_ycbcr422;   // YCbCr 4:2:2
    uint8_t supports_deep_color; // 10/12-bit color
    uint8_t supports_hdr;        // HDR capability
    uint8_t supports_vrr;        // Variable refresh rate
    uint8_t max_tmds_clock;      // Maximum TMDS clock (MHz)
    uint8_t preferred_depth;     // Preferred color depth
} display_caps_t;

display_caps_t detect_display_capabilities(uint8_t *edid) {
    display_caps_t caps = {0};
    
    // Check for HDMI capability (CEA extension block)
    if (edid[126] > 0) {  // Extension block present
        uint8_t *cea_block = edid + 128;
        
        if (cea_block[0] == 0x02) {  // CEA-861 extension
            // Parse CEA data blocks
            uint8_t dtd_offset = cea_block[2];
            uint8_t *data_block_ptr = cea_block + 4;
            
            while (data_block_ptr < cea_block + dtd_offset) {
                uint8_t tag = (*data_block_ptr & 0xE0) >> 5;
                uint8_t length = *data_block_ptr & 0x1F;
                
                switch (tag) {
                    case 1:  // Audio data block
                        caps.supports_audio = 1;
                        break;
                        
                    case 2:  // Video data block
                        // Check for YCbCr support
                        if (cea_block[3] & 0x20) caps.supports_ycbcr444 = 1;
                        if (cea_block[3] & 0x10) caps.supports_ycbcr422 = 1;
                        break;
                        
                    case 3:  // Vendor specific data block
                        // Check for HDMI VSDB (IEEE OUI: 0x000C03)
                        if (length >= 5 && 
                            data_block_ptr[1] == 0x03 &&
                            data_block_ptr[2] == 0x0C &&
                            data_block_ptr[3] == 0x00) {
                            caps.supports_hdmi = 1;
                            
                            // Extract maximum TMDS clock
                            if (length >= 7) {
                                caps.max_tmds_clock = data_block_ptr[7] * 5;
                            }
                            
                            // Check for deep color support
                            if (length >= 6) {
                                uint8_t deep_color = data_block_ptr[6];
                                caps.supports_deep_color = (deep_color & 0x70) != 0;
                            }
                        }
                        break;
                }
                
                data_block_ptr += length + 1;
            }
        }
    }
    
    return caps;
}
```

## Video Standards and Compliance

### CEA-861 Standard Implementation

The system implements CEA-861 video timing standards:

```cpp
// CEA-861 standard video modes
typedef struct {
    uint8_t vic;                 // Video Identification Code
    const char *name;
    uint16_t width, height;
    uint16_t h_total, v_total;
    uint32_t pixel_clock;
    uint8_t progressive;
    uint8_t aspect_ratio;        // 0=4:3, 1=16:9
} cea_timing_t;

static const cea_timing_t cea_modes[] = {
    {1,  "640x480p60",    640,  480,  800,  525,  25175000, 1, 0},
    {2,  "720x480p60",    720,  480,  858,  525,  27000000, 1, 0},
    {3,  "720x480p60",    720,  480,  858,  525,  27000000, 1, 1},
    {4,  "1280x720p60",   1280, 720,  1650, 750,  74250000, 1, 1},
    {5,  "1920x1080i60",  1920, 1080, 2200, 1125, 74250000, 0, 1},
    {16, "1920x1080p60",  1920, 1080, 2200, 1125, 148500000, 1, 1},
    {31, "1920x1080p50",  1920, 1080, 2640, 1125, 148500000, 1, 1},
    {32, "1920x1080p24",  1920, 1080, 2750, 1125, 74250000, 1, 1},
    {34, "1920x1080p30",  1920, 1080, 2200, 1125, 74250000, 1, 1},
    {95, "3840x2160p30",  3840, 2160, 4400, 2250, 297000000, 1, 1},
    {97, "3840x2160p60",  3840, 2160, 4400, 2250, 594000000, 1, 1},
};

// Find best matching CEA mode
int find_cea_mode(uint16_t width, uint16_t height, float refresh) {
    for (int i = 0; i < sizeof(cea_modes)/sizeof(cea_modes[0]); i++) {
        const cea_timing_t *mode = &cea_modes[i];
        
        if (mode->width == width && mode->height == height) {
            float mode_refresh = (float)mode->pixel_clock / 
                               (mode->h_total * mode->v_total);
            
            if (fabs(mode_refresh - refresh) < 1.0f) {
                return mode->vic;
            }
        }
    }
    return 0;  // No matching CEA mode
}
```

### VESA GTF/CVT Implementation

Support for VESA Generalized Timing Formula and Coordinated Video Timings:

```cpp
// VESA GTF calculation
video_mode_t calculate_gtf_mode(uint16_t width, uint16_t height, float refresh) {
    video_mode_t mode = {0};
    
    // GTF constants
    const float MARGIN = 1.8f;           // % margin
    const float CELL_GRAN = 8.0f;        // Character cell granularity
    const float MIN_PORCH = 1.0f;        // Minimum front porch (µs)
    const float V_SYNC_RQD = 3.0f;       // Vertical sync width (lines)
    const float H_SYNC = 8.0f;           // Horizontal sync (% of line time)
    const float MIN_VSYNC_BP = 550.0f;   // Minimum vertical sync + back porch (µs)
    const float M = 600.0f;              // Blanking formula gradient
    const float C = 40.0f;               // Blanking formula offset
    const float K = 128.0f;              // Blanking formula scaling factor
    const float J = 20.0f;               // Blanking formula weighting
    
    // Calculate ideal total lines per frame
    float ideal_duty_cycle = 100.0f - C - (M/height);
    float ideal_h_period = ((100.0f - H_SYNC) / ideal_duty_cycle) * 
                          (MIN_VSYNC_BP / 1000000.0f) / refresh;
    
    // Calculate actual horizontal frequency
    uint16_t v_lines_rnd = height + round(MIN_VSYNC_BP / (ideal_h_period * 1000000.0f));
    float actual_h_freq = 1.0f / ideal_h_period;
    
    // Calculate pixel clock
    float ideal_pixel_clock = actual_h_freq * width * 
                             (100.0f + MARGIN) / 100.0f;
    
    // Round to nearest 0.25 MHz
    uint32_t pixel_clock = (uint32_t)(ideal_pixel_clock / 250000.0f + 0.5f) * 250000;
    
    // Calculate final timings
    mode.width = width;
    mode.height = height;
    mode.pixel_clock = pixel_clock;
    
    // Calculate horizontal timings
    float h_period = 1000000.0f / actual_h_freq;  // µs
    float h_sync_width = round(H_SYNC * h_period / 100.0f / 
                              (1000000.0f / pixel_clock));
    float h_front_porch = round(MIN_PORCH / (1000000.0f / pixel_clock));
    
    mode.h_total = round((float)pixel_clock / actual_h_freq);
    mode.h_sync_start = width + (uint16_t)h_front_porch;
    mode.h_sync_end = mode.h_sync_start + (uint16_t)h_sync_width;
    
    // Calculate vertical timings
    mode.v_total = v_lines_rnd;
    mode.v_sync_start = height + 1;  // Front porch = 1 line
    mode.v_sync_end = mode.v_sync_start + (uint16_t)V_SYNC_RQD;
    
    // Default polarities for GTF
    mode.h_pol = 0;  // Negative
    mode.v_pol = 1;  // Positive
    
    snprintf(mode.name, sizeof(mode.name), "%dx%d@%.1f_GTF", 
             width, height, refresh);
    
    return mode;
}
```

## Advanced Features

### Variable Refresh Rate (VRR) Implementation

VRR support for modern gaming displays:

```cpp
// VRR capability structure
typedef struct {
    uint8_t supported;           // Display supports VRR
    uint8_t active;             // Currently enabled
    uint8_t method;             // 0=HDMI VRR, 1=FreeSync, 2=G-Sync
    uint16_t min_refresh;       // Minimum refresh rate (Hz)
    uint16_t max_refresh;       // Maximum refresh rate (Hz)
    uint32_t current_vtotal;    // Current vertical total
} vrr_state_t;

static vrr_state_t vrr = {0};

// VRR timing adjustment
void update_vrr_timing(float target_refresh) {
    if (!vrr.supported || !vrr.active) return;
    
    // Clamp to supported range
    if (target_refresh < vrr.min_refresh) target_refresh = vrr.min_refresh;
    if (target_refresh > vrr.max_refresh) target_refresh = vrr.max_refresh;
    
    // Calculate new vertical total
    video_mode_t *current_mode = get_current_video_mode();
    uint32_t new_vtotal = (uint32_t)(current_mode->pixel_clock / 
                         (current_mode->h_total * target_refresh));
    
    // Validate timing constraints
    uint32_t min_vtotal = current_mode->height + 10;  // Minimum blanking
    uint32_t max_vtotal = 4095;                       // Hardware limit
    
    if (new_vtotal >= min_vtotal && new_vtotal <= max_vtotal) {
        // Apply new timing
        fpga_core_write(VIDEO_TIMING_BASE + 0x08, new_vtotal << 16);
        vrr.current_vtotal = new_vtotal;
        
        // Notify display via HDMI info frame
        send_vrr_info_frame(target_refresh);
    }
}

// VRR info frame transmission
void send_vrr_info_frame(float refresh_rate) {
    uint8_t info_frame[28] = {0};
    
    // Vendor-specific info frame header
    info_frame[0] = 0x81;  // Vendor-specific info frame
    info_frame[1] = 0x01;  // Version
    info_frame[2] = 0x1B;  // Length
    
    // AMD FreeSync signature
    info_frame[4] = 0x1A;  // IEEE OUI byte 0
    info_frame[5] = 0x00;  // IEEE OUI byte 1  
    info_frame[6] = 0x00;  // IEEE OUI byte 2
    
    // FreeSync data
    info_frame[7] = 0x01;  // FreeSync supported
    info_frame[8] = 0x01;  // FreeSync enabled
    info_frame[9] = vrr.min_refresh;
    info_frame[10] = vrr.max_refresh;
    
    // Calculate checksum
    uint8_t checksum = 0;
    for (int i = 0; i < 28; i++) {
        checksum += info_frame[i];
    }
    info_frame[3] = 0x100 - checksum;
    
    // Send to HDMI encoder
    hdmi_send_info_frame(info_frame, sizeof(info_frame));
}
```

### HDR (High Dynamic Range) Support

Basic HDR metadata transmission:

```cpp
// HDR static metadata structure
typedef struct {
    uint8_t eotf;                // Electro-Optical Transfer Function
    uint8_t metadata_id;         // Static metadata descriptor ID
    uint16_t display_primaries_x[3];  // Red, Green, Blue x coordinates
    uint16_t display_primaries_y[3];  // Red, Green, Blue y coordinates
    uint16_t white_point_x;      // White point x coordinate
    uint16_t white_point_y;      // White point y coordinate
    uint16_t max_luminance;      // Maximum display luminance
    uint16_t min_luminance;      // Minimum display luminance
    uint16_t max_cll;            // Maximum Content Light Level
    uint16_t max_fall;           // Maximum Frame Average Light Level
} hdr_metadata_t;

// Send HDR info frame
void send_hdr_info_frame(hdr_metadata_t *hdr) {
    uint8_t info_frame[30] = {0};
    
    // Dynamic Range and Mastering info frame
    info_frame[0] = 0x87;  // HDR info frame type
    info_frame[1] = 0x01;  // Version
    info_frame[2] = 0x1A;  // Length
    
    // HDR data
    info_frame[4] = hdr->eotf;
    info_frame[5] = hdr->metadata_id;
    
    // Display primaries (little-endian 16-bit values)
    for (int i = 0; i < 3; i++) {
        info_frame[6 + i*4] = hdr->display_primaries_x[i] & 0xFF;
        info_frame[7 + i*4] = (hdr->display_primaries_x[i] >> 8) & 0xFF;
        info_frame[8 + i*4] = hdr->display_primaries_y[i] & 0xFF;
        info_frame[9 + i*4] = (hdr->display_primaries_y[i] >> 8) & 0xFF;
    }
    
    // White point
    info_frame[18] = hdr->white_point_x & 0xFF;
    info_frame[19] = (hdr->white_point_x >> 8) & 0xFF;
    info_frame[20] = hdr->white_point_y & 0xFF;
    info_frame[21] = (hdr->white_point_y >> 8) & 0xFF;
    
    // Luminance values
    info_frame[22] = hdr->max_luminance & 0xFF;
    info_frame[23] = (hdr->max_luminance >> 8) & 0xFF;
    info_frame[24] = hdr->min_luminance & 0xFF;
    info_frame[25] = (hdr->min_luminance >> 8) & 0xFF;
    info_frame[26] = hdr->max_cll & 0xFF;
    info_frame[27] = (hdr->max_cll >> 8) & 0xFF;
    info_frame[28] = hdr->max_fall & 0xFF;
    info_frame[29] = (hdr->max_fall >> 8) & 0xFF;
    
    // Calculate checksum
    uint8_t checksum = 0;
    for (int i = 0; i < 30; i++) {
        checksum += info_frame[i];
    }
    info_frame[3] = 0x100 - checksum;
    
    // Send to HDMI encoder
    hdmi_send_info_frame(info_frame, sizeof(info_frame));
}
```

### Game Mode Optimizations

Gaming-specific optimizations for minimal latency:

```cpp
// Game mode configuration
typedef struct {
    uint8_t low_latency_mode;    // Minimize processing delays
    uint8_t auto_low_latency;    // Automatic switching
    uint8_t direct_video;        // Bypass scaler
    uint8_t variable_refresh;    // Enable VRR
    uint8_t reduced_blanking;    // CVT-RB timings
} game_mode_config_t;

void enable_game_mode(game_mode_config_t *config) {
    if (config->low_latency_mode) {
        // Reduce video buffer depth
        fpga_core_write(SCALER_CONFIG_BASE + 0x00, 0x01);  // Single buffer
        
        // Enable direct video path
        if (config->direct_video) {
            fpga_core_write(VIDEO_MUX_BASE, 0x02);  // Bypass scaler
        }
        
        // Minimize audio latency
        fpga_core_write(AUDIO_CONFIG_BASE + 0x04, 0x10);  // Small audio buffer
        
        // Enable VRR if supported
        if (config->variable_refresh && vrr.supported) {
            vrr.active = 1;
            update_vrr_timing(60.0f);  // Default to 60Hz
        }
    }
    
    // Send Auto Low Latency Mode signal to display
    if (config->auto_low_latency) {
        send_allm_info_frame(1);
    }
}

void send_allm_info_frame(uint8_t enable) {
    uint8_t info_frame[8] = {0};
    
    // Vendor-specific info frame for ALLM
    info_frame[0] = 0x81;  // Vendor-specific
    info_frame[1] = 0x01;  // Version
    info_frame[2] = 0x04;  // Length
    
    // HDMI LLC OUI (0x0000F0)
    info_frame[4] = 0xF0;
    info_frame[5] = 0x00;
    info_frame[6] = 0x00;
    
    // ALLM data
    info_frame[7] = enable ? 0x01 : 0x00;
    
    // Calculate checksum
    uint8_t checksum = 0;
    for (int i = 0; i < 8; i++) {
        checksum += info_frame[i];
    }
    info_frame[3] = 0x100 - checksum;
    
    hdmi_send_info_frame(info_frame, sizeof(info_frame));
}
```

## Integration and Synchronization

### Multi-Clock Domain Synchronization

The HDMI system operates across multiple clock domains requiring careful synchronization:

```systemverilog
// Clock domain definitions
wire clk_core;          // Core video clock (variable)
wire clk_hdmi;          // HDMI pixel clock (PLL generated)
wire clk_audio;         // Audio clock (fixed ratios)
wire clk_sys;           // System clock (100 MHz)

// Synchronization components
async_fifo #(
    .DATA_WIDTH(24),    // RGB data
    .ADDR_WIDTH(10)     // 1K FIFO
) video_sync_fifo (
    .wr_clk(clk_core),
    .wr_data({core_r, core_g, core_b}),
    .wr_en(core_de),
    
    .rd_clk(clk_hdmi),
    .rd_data({hdmi_r, hdmi_g, hdmi_b}),
    .rd_en(hdmi_de)
);

// Audio synchronization
audio_sync #(
    .SAMPLE_WIDTH(16)
) audio_sync_inst (
    .clk_audio(clk_audio),
    .clk_sys(clk_sys),
    
    .audio_l_in(core_audio_l),
    .audio_r_in(core_audio_r),
    .sample_valid_in(core_audio_valid),
    
    .audio_l_out(hdmi_audio_l),
    .audio_r_out(hdmi_audio_r),
    .sample_valid_out(hdmi_audio_valid)
);
```

### Frame Synchronization

Ensures proper A/V sync across different clock domains:

```systemverilog
// Frame synchronization controller
module frame_sync (
    input  wire clk_video,
    input  wire clk_audio,
    input  wire reset,
    
    // Video timing
    input  wire vsync_in,
    input  wire hsync_in,
    
    // Audio timing  
    input  wire audio_sample_clk,
    
    // Synchronized outputs
    output reg  vsync_out,
    output reg  audio_sync_pulse,
    
    // Status
    output wire sync_locked
);

// Phase detector for A/V sync
reg [15:0] video_counter;
reg [15:0] audio_counter;
reg [7:0]  sync_error;

always @(posedge clk_video) begin
    if (vsync_in) begin
        video_counter <= 0;
        
        // Compare with audio timing
        if (audio_counter > video_counter) begin
            sync_error <= audio_counter - video_counter;
        end else begin
            sync_error <= video_counter - audio_counter;
        end
    end else begin
        video_counter <= video_counter + 1;
    end
end

always @(posedge clk_audio) begin
    if (audio_sample_clk) begin
        audio_counter <= audio_counter + 1;
    end
    
    if (vsync_in) begin
        audio_counter <= 0;
        audio_sync_pulse <= 1;
    end else begin
        audio_sync_pulse <= 0;
    end
end

assign sync_locked = (sync_error < 8);  // Within acceptable range
endmodule
```

## Performance and Optimization

### Bandwidth Optimization

Optimizing memory bandwidth for high-resolution video:

```cpp
// Bandwidth calculation and optimization
typedef struct {
    uint32_t pixel_rate;        // Pixels per second
    uint8_t  bits_per_pixel;    // Color depth
    uint32_t bandwidth_required; // Bytes per second
    uint32_t bandwidth_available; // Available bandwidth
    uint8_t  compression_ratio;  // If compression enabled
} bandwidth_calc_t;

bandwidth_calc_t calculate_bandwidth(video_mode_t *mode) {
    bandwidth_calc_t calc = {0};
    
    calc.pixel_rate = mode->pixel_clock;
    calc.bits_per_pixel = 24;  // RGB888
    
    // Raw bandwidth requirement
    calc.bandwidth_required = (calc.pixel_rate * calc.bits_per_pixel) / 8;
    
    // Available DDR3 bandwidth (theoretical)
    calc.bandwidth_available = 800000000;  // 800 MHz DDR3
    calc.bandwidth_available *= 8;         // 64-bit interface
    calc.bandwidth_available = calc.bandwidth_available * 0.8;  // 80% efficiency
    
    // Check if compression needed
    if (calc.bandwidth_required > calc.bandwidth_available * 0.3) {
        calc.compression_ratio = 2;  // 2:1 compression
    } else {
        calc.compression_ratio = 1;  // No compression
    }
    
    printf("Video bandwidth: %u MB/s (%.1f%% of available)\n",
           calc.bandwidth_required / 1000000,
           (float)calc.bandwidth_required / calc.bandwidth_available * 100.0f);
    
    return calc;
}
```

### Latency Optimization

Minimizing end-to-end video latency:

```cpp
// Latency measurement and optimization
typedef struct {
    uint32_t input_capture_latency;    // Core to FIFO
    uint32_t processing_latency;       // Video processing
    uint32_t output_latency;          // FIFO to HDMI
    uint32_t display_latency;         // Display processing
    uint32_t total_latency;           // End-to-end
} latency_profile_t;

latency_profile_t measure_video_latency() {
    latency_profile_t profile = {0};
    
    // Measure using test pattern timing
    fpga_core_write(TEST_PATTERN_BASE, 0x01);  // Enable test pattern
    
    // Capture timestamps at various pipeline stages
    uint32_t t0 = fpga_core_read(TIMESTAMP_BASE + 0x00);  // Input
    uint32_t t1 = fpga_core_read(TIMESTAMP_BASE + 0x04);  // Post-processing
    uint32_t t2 = fpga_core_read(TIMESTAMP_BASE + 0x08);  // Output
    
    // Calculate latencies (in pixel clocks)
    profile.input_capture_latency = t1 - t0;
    profile.processing_latency = t2 - t1;
    profile.output_latency = 10;  // Estimated HDMI encoder delay
    
    // Convert to milliseconds
    float pixel_period = 1000000.0f / get_current_pixel_clock();
    profile.total_latency = (profile.input_capture_latency + 
                           profile.processing_latency + 
                           profile.output_latency) * pixel_period;
    
    return profile;
}

// Latency optimization strategies
void optimize_latency() {
    // Enable direct video mode (bypass scaler)
    fpga_core_write(VIDEO_MUX_BASE, 0x02);
    
    // Reduce FIFO depths
    fpga_core_write(VIDEO_FIFO_CONFIG, 0x04);  // Minimum buffering
    
    // Enable fast pixel clock switching
    fpga_core_write(PLL_CONFIG_BASE + 0x10, 0x01);
    
    // Disable unnecessary processing
    fpga_core_write(VIDEO_EFFECTS_BASE, 0x00);  // No effects
}
```

## Development and Debugging

### Video Signal Analysis

Built-in tools for video signal debugging:

```cpp
// Video signal analyzer
typedef struct {
    uint32_t h_total_measured;
    uint32_t v_total_measured;
    uint32_t pixel_clock_measured;
    float    refresh_rate_measured;
    uint8_t  sync_polarities;
    uint8_t  signal_stable;
} video_analysis_t;

video_analysis_t analyze_video_signal() {
    video_analysis_t analysis = {0};
    
    // Enable measurement counters
    fpga_core_write(VIDEO_ANALYZER_BASE + 0x00, 0x01);
    
    // Wait for measurement period
    usleep(100000);  // 100ms
    
    // Read measurements
    analysis.h_total_measured = fpga_core_read(VIDEO_ANALYZER_BASE + 0x04);
    analysis.v_total_measured = fpga_core_read(VIDEO_ANALYZER_BASE + 0x08);
    analysis.pixel_clock_measured = fpga_core_read(VIDEO_ANALYZER_BASE + 0x0C);
    
    // Calculate refresh rate
    if (analysis.h_total_measured && analysis.v_total_measured) {
        analysis.refresh_rate_measured = 
            (float)analysis.pixel_clock_measured / 
            (analysis.h_total_measured * analysis.v_total_measured);
    }
    
    // Check signal stability
    uint32_t stability_count = fpga_core_read(VIDEO_ANALYZER_BASE + 0x10);
    analysis.signal_stable = (stability_count > 95);  // 95% stable frames
    
    // Read sync polarities
    analysis.sync_polarities = fpga_core_read(VIDEO_ANALYZER_BASE + 0x14) & 0x03;
    
    return analysis;
}

// Debug output function
void print_video_debug_info() {
    video_analysis_t analysis = analyze_video_signal();
    
    printf("=== Video Signal Analysis ===\n");
    printf("Horizontal Total: %u pixels\n", analysis.h_total_measured);
    printf("Vertical Total: %u lines\n", analysis.v_total_measured);
    printf("Pixel Clock: %.3f MHz\n", analysis.pixel_clock_measured / 1000000.0f);
    printf("Refresh Rate: %.2f Hz\n", analysis.refresh_rate_measured);
    printf("H Sync Polarity: %s\n", (analysis.sync_polarities & 1) ? "Positive" : "Negative");
    printf("V Sync Polarity: %s\n", (analysis.sync_polarities & 2) ? "Positive" : "Negative");
    printf("Signal Stable: %s\n", analysis.signal_stable ? "Yes" : "No");
}
```

### Test Pattern Generation

Built-in test patterns for validation:

```systemverilog
// Test pattern generator
module test_pattern_gen (
    input  wire        clk,
    input  wire        reset,
    input  wire        enable,
    
    // Timing inputs
    input  wire [11:0] h_count,
    input  wire [11:0] v_count,
    input  wire        data_enable,
    
    // Pattern selection
    input  wire [3:0]  pattern_select,
    
    // RGB output
    output reg [7:0]   r_out,
    output reg [7:0]   g_out,
    output reg [7:0]   b_out
);

always @(posedge clk) begin
    if (reset) begin
        r_out <= 0;
        g_out <= 0;
        b_out <= 0;
    end else if (enable && data_enable) begin
        case (pattern_select)
            4'h0: begin  // Color bars
                case (h_count[10:8])
                    3'h0: {r_out, g_out, b_out} <= 24'hFFFFFF;  // White
                    3'h1: {r_out, g_out, b_out} <= 24'hFFFF00;  // Yellow
                    3'h2: {r_out, g_out, b_out} <= 24'h00FFFF;  // Cyan
                    3'h3: {r_out, g_out, b_out} <= 24'h00FF00;  // Green
                    3'h4: {r_out, g_out, b_out} <= 24'hFF00FF;  // Magenta
                    3'h5: {r_out, g_out, b_out} <= 24'hFF0000;  // Red
                    3'h6: {r_out, g_out, b_out} <= 24'h0000FF;  // Blue
                    3'h7: {r_out, g_out, b_out} <= 24'h000000;  // Black
                endcase
            end
            
            4'h1: begin  // Gradient
                r_out <= h_count[7:0];
                g_out <= v_count[7:0];
                b_out <= h_count[7:0] ^ v_count[7:0];
            end
            
            4'h2: begin  // Checkerboard
                if ((h_count[6] ^ v_count[6]) == 1) begin
                    {r_out, g_out, b_out} <= 24'hFFFFFF;
                end else begin
                    {r_out, g_out, b_out} <= 24'h000000;
                end
            end
            
            4'h3: begin  // Grid
                if (h_count[6:0] == 0 || v_count[6:0] == 0) begin
                    {r_out, g_out, b_out} <= 24'hFFFFFF;
                end else begin
                    {r_out, g_out, b_out} <= 24'h404040;
                end
            end
            
            default: begin
                r_out <= 8'h80;
                g_out <= 8'h80;
                b_out <= 8'h80;
            end
        endcase
    end else begin
        r_out <= 0;
        g_out <= 0;
        b_out <= 0;
    end
end
endmodule
```

This comprehensive HDMI documentation provides developers with detailed technical knowledge for understanding, integrating, and optimizing the MiSTer HDMI system for professional video output applications.

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"id": "1", "content": "Analyze HDMI hardware interfaces and drivers", "status": "completed", "priority": "high"}, {"id": "2", "content": "Document HDMI timing and configuration", "status": "completed", "priority": "high"}, {"id": "3", "content": "Examine HDMI audio integration", "status": "completed", "priority": "high"}, {"id": "4", "content": "Document HDMI PLL and clock management", "status": "completed", "priority": "high"}, {"id": "5", "content": "Analyze HDMI control from ARM side", "status": "completed", "priority": "medium"}, {"id": "6", "content": "Create comprehensive HDMI documentation", "status": "completed", "priority": "high"}]