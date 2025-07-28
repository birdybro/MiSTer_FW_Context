# MiSTer Video Freak Module - Comprehensive Documentation

## Table of Contents
1. [Overview](#overview)
2. [Module Architecture](#module-architecture)
3. [Video Cropping System](#video-cropping-system)
4. [Integer Scaling System](#integer-scaling-system)
5. [Mathematical Operations](#mathematical-operations)
6. [Aspect Ratio Calculation](#aspect-ratio-calculation)
7. [Signal Flow](#signal-flow)
8. [Configuration Parameters](#configuration-parameters)
9. [Integration with MiSTer Framework](#integration-with-mister-framework)
10. [Technical Implementation Details](#technical-implementation-details)

## Overview

The Video Freak module is a sophisticated video processing system in the MiSTer FPGA framework that provides advanced video cropping and integer scaling capabilities. It enables precise control over video output geometry, aspect ratio correction, and scaling modes to optimize display quality across different resolutions and display devices.

### Key Features
- **Dynamic Video Cropping**: Real-time video area selection and offset adjustment
- **Integer Scaling**: Multiple integer scaling modes for pixel-perfect display
- **Aspect Ratio Correction**: Automatic aspect ratio calculation with crop compensation
- **Multi-Mode Scaling**: Support for vertical-only, horizontal-vertical, and adaptive scaling
- **Real-time Processing**: All operations performed in video clock domain

### Authors and License
- Video Crop: Copyright (c) 2020 Grabulosaure, (c) 2021 Alexey Melnikov
- Integer Scaling: Copyright (c) 2021 Alexey Melnikov
- Licensed under GPL

## Module Architecture

The video_freak system consists of two primary modules:

### 1. video_freak (Main Module)
- **Purpose**: Video cropping and aspect ratio management
- **File**: `Template_MiSTer-master/sys/video_freak.sv:16-138`
- **Clock Domain**: CLK_VIDEO with CE_PIXEL enable

### 2. video_scale_int (Scaling Submodule)
- **Purpose**: Integer scaling calculations and mode handling
- **File**: `Template_MiSTer-master/sys/video_freak.sv:141-329`
- **Integration**: Instantiated within main video_freak module

## Video Cropping System

### Input Processing and Frame Detection

The cropping system performs real-time analysis of incoming video signals:

```systemverilog
// Frame dimension detection (lines 55-71)
if (CE_PIXEL) begin
    old_de <= VGA_DE_IN;
    old_vs <= VGA_VS;
    if (VGA_VS & ~old_vs) begin
        vcpt  <= 0;
        vtot  <= vcpt;        // Total vertical lines
        vcalc <= 1;
        vcrop <= (CROP_SIZE >= vcpt) ? 12'd0 : CROP_SIZE;
    end
    
    if (VGA_DE_IN) hcpt <= hcpt + 1'd1;    // Count horizontal pixels
    if (~VGA_DE_IN & old_de) begin
        vcpt <= vcpt + 1'd1;               // Count vertical lines
        if(!vcpt) hsize <= hcpt;           // Capture horizontal size
        hcpt <= 0;
    end
end
```

### Crop Size Validation
- **Range Check**: Ensures CROP_SIZE doesn't exceed frame height
- **Fallback**: Disables cropping if CROP_SIZE >= frame height
- **Dynamic Update**: Recalculates on every frame

### Vertical Offset Calculation

Advanced offset calculation with signed arithmetic (lines 116-118):

```systemverilog
vadj <= (vtot-vcrop) + {{6{CROP_OFF[4]}},CROP_OFF,1'b0};
voff <= vadj[11] ? 12'd0 : ((vadj[11:1] + vcrop) > vtot) ? vtot-vcrop : vadj[11:1];
```

- **CROP_OFF Range**: -16 to +15 (5-bit signed)
- **Scaling**: Multiplied by 2 for pixel-level precision
- **Boundary Protection**: Ensures offset doesn't exceed frame boundaries

### Active Area Generation

```systemverilog
ovde <= ((vcpt >= voff) && (vcpt < (vcrop + voff))) || !vcrop;
vde  <= ovde;
assign VGA_DE = vde & VGA_DE_IN;
```

## Integer Scaling System

### Scaling Modes

The system supports five distinct scaling modes (SCALE parameter):

| Mode | Description | Use Case |
|------|-------------|----------|
| 0 | Normal (No scaling) | Direct aspect ratio pass-through |
| 1 | V-Integer | Vertical integer scaling only |
| 2 | HV-Integer- | Conservative horizontal-vertical integer |
| 3 | HV-Integer+ | Adaptive horizontal-vertical integer |
| 4 | HV-Integer | Optimal horizontal-vertical integer |

### Mathematical Pipeline

The scaling calculation follows a sophisticated 14-stage pipeline:

#### Stage 0-1: Vertical Scale Factor
```systemverilog
div_num <= HDMI_HEIGHT;    // e.g., 1080
div_den <= vsize;          // e.g., 400
// Result: 1080/400 = 2 (integer scale factor)
```

#### Stage 2-3: Scaled Height Calculation
```systemverilog
mul_arg1 <= vsize;         // e.g., 400
mul_arg2 <= div_res[11:0]; // e.g., 2
// Result: 400 * 2 = 800 (scaled height)
```

#### Stage 4-5: Aspect Ratio Target Width
```systemverilog
mul_arg1 <= mul_res[11:0]; // e.g., 800
mul_arg2 <= arx_i;         // e.g., 4
div_den  <= ary_i;         // e.g., 3
// Result: (800 * 4) / 3 = 1066 (target width)
```

### Example Calculations

For a 720x400 4:3 source on 1920x1080 display:

1. **Vertical Scale**: 1080 ÷ 400 = 2.7 → 2 (integer)
2. **Scaled Height**: 400 × 2 = 800 pixels
3. **Target Width**: (800 × 4) ÷ 3 = 1066 pixels
4. **Horizontal Scale**: 1920 ÷ 720 = 2.67 → 2 (integer)
5. **Scaled Width**: 720 × 2 = 1440 pixels

## Mathematical Operations

The module utilizes two custom mathematical units from `math.sv`:

### sys_umul (Unsigned Multiplier)
- **Purpose**: 12-bit × 12-bit → 24-bit multiplication
- **Implementation**: Serial bit-shift algorithm
- **Latency**: Variable (depends on operand values)
- **Usage**: Aspect ratio and scaling calculations

### sys_udiv (Unsigned Divider)
- **Purpose**: 24-bit ÷ 12-bit → 24-bit division
- **Algorithm**: Non-restoring division
- **Latency**: Fixed NB_NUM + 2 cycles
- **Features**: Provides both quotient and remainder

### Pipeline Control
```systemverilog
if(~div_start & ~div_run & ~mul_start & ~mul_run) begin
    cnt <= cnt + 1'd1;  // Advance to next calculation stage
```

## Aspect Ratio Calculation

### Crop-Compensated Aspect Ratio

When cropping is active, the module recalculates aspect ratios to maintain correct proportions:

```systemverilog
// Multiplication stages in vcalc state machine (lines 88-104)
case(vcalc)
    1: begin
        mul_arg1  <= arx;    // Original aspect X
        mul_arg2  <= vtot;   // Total frame height
        mul_start <= 1;
    end
    2: begin
        ARXG      <= mul_res;  // ARX × vtot
        mul_arg1  <= ary;      // Original aspect Y
        mul_arg2  <= vcrop;    // Cropped height
        mul_start <= 1;
    end
    3: begin
        ARYG      <= mul_res;  // ARY × vcrop
    end
endcase
```

### Normalization Process
```systemverilog
// Bit-shifting normalization (lines 107-114)
if (ARXG[23] | ARYG[23]) begin
    arxo <= ARXG[23:12];  // Take upper 12 bits
    aryo <= ARYG[23:12];
end
else begin
    ARXG <= ARXG << 1;    // Left-shift until normalized
    ARYG <= ARYG << 1;
end
```

## Signal Flow

### Input Signals
- **CLK_VIDEO**: Primary clock domain
- **CE_PIXEL**: Pixel clock enable
- **VGA_VS**: Vertical sync for frame detection
- **VGA_DE_IN**: Input display enable
- **HDMI_WIDTH/HEIGHT**: Target display resolution
- **ARX/ARY**: Source aspect ratio (12-bit each)
- **CROP_SIZE**: Desired crop height (12-bit)
- **CROP_OFF**: Vertical offset (-16 to +15, 5-bit signed)
- **SCALE**: Scaling mode selection (3-bit)

### Output Signals
- **VGA_DE**: Processed display enable
- **VIDEO_ARX/ARY**: Final aspect ratio (13-bit each)

### Internal Data Flow
```
Input Video → Frame Detection → Crop Calculation → Aspect Correction → Integer Scaling → Output
     ↓              ↓               ↓                ↓                    ↓           ↓
  hsize/vsize → crop validation → Math Pipeline → Ratio Normalization → Mode Selection
```

## Configuration Parameters

### CROP_SIZE (12-bit)
- **Range**: 0 to 4095 lines
- **Function**: Specifies desired crop height
- **Behavior**: 0 or >= frame height disables cropping

### CROP_OFF (5-bit signed)
- **Range**: -16 to +15
- **Function**: Vertical position offset
- **Resolution**: 2-pixel steps (internally doubled)

### SCALE (3-bit)
- **0**: Normal - No integer scaling applied
- **1**: V-Integer - Vertical integer scaling only
- **2**: HV-Integer- - Conservative horizontal-vertical scaling
- **3**: HV-Integer+ - Adaptive scaling with screen utilization priority
- **4**: HV-Integer - Optimal scaling balancing accuracy and screen usage

### ARX/ARY (12-bit each)
- **Range**: 0 to 4095
- **Function**: Source material aspect ratio
- **Special**: ARY=0 triggers horizontal-only scaling mode

## Integration with MiSTer Framework

### sys_top.v Integration
The video_freak module is typically instantiated in the video processing chain:

```systemverilog
video_freak video_freak_inst (
    .CLK_VIDEO(CLK_VIDEO),
    .CE_PIXEL(CE_PIXEL),
    .VGA_VS(VGA_VS),
    .HDMI_WIDTH(HDMI_WIDTH),
    .HDMI_HEIGHT(HDMI_HEIGHT),
    .VGA_DE(VGA_DE_CROP),
    .VIDEO_ARX(VIDEO_ARX),
    .VIDEO_ARY(VIDEO_ARY),
    
    .VGA_DE_IN(VGA_DE_RAW),
    .ARX(status[12:1]),
    .ARY(status[24:13]),
    .CROP_SIZE(crop_size),
    .CROP_OFF(crop_offset),
    .SCALE(scale_mode)
);
```

### Configuration Interface
- **HPS Communication**: Parameters typically set via ARM processor
- **Real-time Updates**: All parameters can be changed during operation
- **Status Integration**: Often controlled through MiSTer menu system

## Technical Implementation Details

### Clock Domain Considerations
- **Single Clock Domain**: All operations in CLK_VIDEO domain
- **CE_PIXEL Gating**: Ensures synchronization with pixel rate
- **Pipeline Depth**: Mathematical operations span multiple clock cycles

### Memory Requirements
- **No External Memory**: All calculations performed in registers
- **Register Usage**: Approximately 200-300 flip-flops
- **DSP Blocks**: None required (pure logic implementation)

### Timing Characteristics
- **Input Latency**: 1-2 pixel clock cycles
- **Calculation Latency**: 14+ clock cycles for scaling computation
- **Throughput**: Full pixel rate after initial calculation

### Error Handling
- **Division by Zero**: Protected by conditional checks
- **Overflow Protection**: 24-bit intermediate values prevent overflow
- **Boundary Checking**: Prevents invalid crop/offset combinations

### Performance Optimization
- **Pipeline Stages**: Calculations spread across multiple cycles
- **Early Termination**: Some modes skip unnecessary calculations
- **Bit-Level Operations**: Efficient normalization using shifts

### Compatibility
- **Resolution Range**: Supports 1x1 to 4095x4095 resolutions
- **Aspect Ratios**: Any ratio expressible in 12-bit integers
- **Display Types**: Compatible with all HDMI/VGA output modes

This comprehensive documentation covers the complete functionality of the MiSTer Video Freak module, providing both high-level understanding and detailed implementation specifics for developers working with the MiSTer FPGA framework.