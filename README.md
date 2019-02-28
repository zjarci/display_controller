# Project F Display Controller

The Project F display controller is designed to make it easy to add video output to FPGA projects. It's written in Verilog and supports VGA, DVI, and HDMI displays. It includes full configuration for 640x480, 800x600, 1280x720, and 1920x1080, as well as the ability to define custom resolutions. This design and its documentation are licensed under the MIT License.

For tutorials and further information visit https://projectf.io

## Hardware Interface Support
This design supports VGA, DVI, and HDMI displays. HDMI support is currently via backwards compatibility with DVI. HDMI features, such as audio, will be added in future.

Two flavours of DVI are available: direct TMDS generation and [Black Mesa Labs (BML) DVI Pmod](https://blackmesalabs.wordpress.com/2017/12/15/bml-hdmi-video-for-fpgas-over-pmod/) with Texas Instruments [TFP410](http://www.ti.com/product/TFP410). Direct TMDS generation on FPGA requires high-frequency clocks and SerDes, but allows full control of the signal including HDMI features. The BML Pmod is restricted to standard DVI features, but allows even tiny FPGAs to support DVI signalling.

As of February 2019 there is an unresolved issue when using the BML DVI Pmods with HDMI displays: some displays report signal error and show nothing. This issue isn't confined to Project F, but has also been reported by Black Mesa Labs themselves.

## Resolution Support
The following four display resolutions are tested and included by default (all at 60 Hz):
    
     Resolution  Ratio   Clock     
     640 x  480    4:3   25.20 MHz
     800 x  600    4:3   40.00 MHz
    1280 x  720   16:9   74.25 MHz     
    1920 x 1080   16:9  148.50 MHz   

You can easily add your own timings for other resolutions.

_NB. The canonical clock for 640x480 60Hz is 25.175 MHz, but 25.2 MHz is within VESA spec and easier to generate._

## Demos
The [demo](hdl/demo) directory includes demos for each of the supported interfaces:

* [`display_demo_dvi.v`](hdl/demo/display_demo_dvi.v) - DVI encoded on FPGA
* [`display_demo_dvi_pmod3.v`](hdl/demo/display_demo_dvi_pmod3.v) - DVI generated by BML 3-bit Pmod
* `display_demo_dvi_pmod24.v` - DVI generated by BML 24-bit Pmod (available soon)
* [`display_demo_vga.v`](hdl/demo/display_demo_vga.v) - analogue VGA with 12-bit output

You can find the list of required modules for each demo in a comment at the top of its file. You'll also need suitable constraints, such as those in Project F [hardware-support](https://github.com/projf/hardware-support).

You can adjust the demo resolution by changing the parameters for `display_clocks`, `display_timings`, and `test_card`. Comments in the demos provide settings for tested resolutions (see above).

## Architecture
There are two different high-level designs depending on whether the FPGA is doing TMDS encoding.

### Analogue VGA and BML DVI Pmod

1. Display Clocks - synthesizes the pixel clock, for example 40 MHz for 800x600
2. Display Timings - generates the display sync signals and active pixel position
3. Colour Data - the colour of a pixel, taken from a bitmap, sprite, test card etc.
4. Parallel Colour Output - external hardware converts this to analogue VGA or TMDS DVI as appropriate

### TMDS Encoding on FPGA for DVI or HDMI

1. Display Clocks - synthesizes the pixel and SerDes clocks, for example 74.25 and 371.25 MHz for 720p
2. Display Timings - generates the display sync signals and active pixel position
3. Colour Data - the colour of a pixel, taken from a bitmap, sprite, test card etc.
4. TMDS Encoder - encodes 8-bit red, green, and blue pixel data into 10-bit TMDS values
5. 10:1 Serializer - converts parallel 10-bit TMDS value into serial form
6. Differential Signal Output - converts the TMDS data into differential form for output via two FPGA pins

## Modules
* **[display_clocks](/display/hdl/display_clocks.v)** - pixel and high-speed clocks for TMDS (includes Xilinx MMCM)
* **[display_timings](/display/hdl/display_timings.v)** - generates display timings, including horizontal and vertical sync
* **[dvi_generator](/display/hdl/dvi_generator.v)** - uses `serializer_10to1` and `tmds_encode_dvi` to generate a DVI signal
* **[serializer_10to1](/display/hdl/serializer_10to1.v)** - serializes the 10-bit TMDS data (includes Xilinx OSERDESE2)
* **[test_card](/display/hdl/test_card.v)** - generates a video test card based on provided resolution
* **[tmds_encoder_dvi](/display/hdl/tmds_encoder_dvi.v)** - encodes 8-bit per colour into 10-bit TMDS values for DVI

There is also a _top_ module for the display controller, [demo](../hdl/demo) versions of which are included with this project. When performing TMDS encoding on FPGA the top module makes use of the Xilinx OBUFDS buffer to generate the differential output signals.

### Module Parameters

#### Display Clocks
* `MULT_MASTER` - multiplication for both output clocks
* `DIV_MASTER` - division for both output clocks
* `DIV_5X` - division for SerDes clock
* `DIV_1X` - division for pixel clock
* `IN_PERIOD` - period of input clock in nanoseconds (ns)

#### Display Timings
* `H_RES` - active horizontal resolution in pixels 
* `V_RES` - active vertical resolution in lines 
* `H_FP` - horizontal front porch length in pixels
* `H_SYNC` - horizontal sync length in pixels
* `H_BP` - horizontal back porch length in pixels
* `V_FP` - vertical front porch length in lines
* `V_SYNC` - vertical sync length in lines
* `V_BP` - vertical back porch length in lines
* `H_POL` - horizontal sync polarity (0:negative, 1:positive)
* `V_POL` - vertical sync polarity (0:negative, 1:positive)

#### Test Card
* `H_RES` - horizontal test card resolution in pixels 
* `V_RES` - vertical test card resolution in lines 

## Porting
We strive to create generic HDL designs where possible, however vendor specific components are critical to certain functionality, such as high-speed clock generation. The display controller uses three Xilinx-specific components: all display options use the `MMCM` for clock generation, while TMDS encoding on the FPGA requires `OSERDESE2` and `OBUFDS`. Versions for other manufacturers will be added in due course, but in the meantime the following advice should help with porting:

* **MMCM** - Mixed-Mode Clock Manager: Replace with the clock or clock synthesizer of your choice, such as PLL. The simplest pixel clock to generate is usually the 40.0 MHz required by 800x600 60 Hz. Some of the pixel clocks are quite hard to generate accurately, which is where the MMCM's support for multiplying and dividing by eighths comes in handy, for example: `74.25 MHz = 100 MHz * 37.125 / 50`. Analogue VGA is relatively tolerant of inaccurate clock frequencies; DVI and HDMI much less so.
* **OSERDESE2** - SerDes: TMDS requires 10:1 serialization, which is supported by chaining two OSERDESE2 together. This is generally the hardest part of the design to port as FPGAs vary enormously in their SerDes capabilities. If native 10:1 serialization isn't available you can sometimes make use of logic designed for DDR memory or divide serialization into two 5-bit steps. Only needed for on-FPGA TMDS generation.
* **OBUFDS** - Differential Signalling: Provided your FPGA has at least four pairs of DS pins you should easily be able to use this design. Only needed for on-FPGA TMDS generation.
