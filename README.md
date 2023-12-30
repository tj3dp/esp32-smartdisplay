# LVGL drivers for Chinese Sunton Smart display boards, aka CYD (Cheap Yellow Display)

[![Platform IO CI](https://github.com/rzeldent/esp32-smartdisplay/actions/workflows/main.yml/badge.svg)](https://github.com/rzeldent/esp32-smartdisplay/actions/workflows/main.yml)

## Supported boards

These Sunton boards have an LCD display and most of them have a touch interface.
More information, data sheets, ordering information etc. can be found at [Sunton Boards information](https://github.com/rzeldent/platformio-espressif32-sunton)

- N = No touch
- R = Resistive touch
- C = Capacitive touch

Currently this library supports the following boards:

- ESP32-1732S019N/C
- ESP32-2424S012N/C
- ESP32-2432S024R/C/N
- ESP32-2432S028R
- ESP32-2432S032N/R/C
- ESP32-3248S035R/C
- ESP32-4827S043R/C
- ESP32-8048S050N/C
- ESP32-8048S070N/C
- ESP32-4848S040C

This library integrates seamlessly in [PlatformIO](https://platformio.org/) and supports these boards with little effort and provides a jump start!

## Why this library

With the boards, there is a link supplied and there are a lot of examples present and this looks fine.... If you know your way around....
These examples for [LVGL](https://lvgl.io/) depend on external libraries ([LCD_eSPI](https://github.com/Bodmer/LCD_eSPI) or [LovyanGFX](https://github.com/lovyan03/LovyanGFX)).
However, when working with these libraries, I found out that these libraries had their flaws using these boards:

- Lots of configuring to do before it all works,
- Using a third party library there is unnecessary code included (for other boards),
- No support for on the fly rotating,
- No support for touch,
- Not using code already present in the ESP firmware
- No LVGL integration

## Dependencies

This library depends on:

- [LVGL](https://registry.platformio.org/libraries/lvgl/lvgl), currently version 8.3.9
- [platformio-espressif32-sunton](https://github.com/rzeldent/platformio-espressif32-sunton)

>[!NOTE]
>This library uses the "official" drivers from Espressif's component service. These drivers use the newly introduced esp_lcd_panel interfaces. This should provide some support in the future for updates and new boards. These drivers have already been copied and included to this library.

## How to use

Get started by following the steps below. It is also highly recommended to look at the demo application [esp32-smartdisplay-demo](https://github.com/rzeldent/esp32-smartdisplay-demo) to quickly see the possibilities of this library.

![Demo screen](assets/images/PXL_20231130_225143662.jpg)

Additionally this demo provides:

- UI created using the SquareLine Studio GUI generator.
- Sound over I2S and internal DAC
- Read the CDR (light sensor)
- Control of the LEDs
- PlatformIO definitions for all the known boards
- Full source code

The next sections will guide you though the process of creating an application. However, some knowledge of PlatformIO, C/C++ and LVGL is required!

If you run into problems, first try to open a discussion on the [github esp32-smartdisplay discussion board](https://github.com/rzeldent/esp32-smartdisplay/discussions)

### Step 1: Download (or open) PlatformIO

[![PlatformIO](assets/images/PlatformIO.png)](https://platformio.org/)

This library is made for usage in PlatformIO. If not familiar with PlatformIO please take a look at their site and micro controller boards they support. However these boards only use the Arduino platform and ESP32 boards.

Make sure you have PlatformIO installed and functional. Follow the documentation on their site:
[https://docs.platformio.org/en/latest/](https://docs.platformio.org/en/latest/)

The board definitions required for this library are defined in the boards library platformio-espressif32-sunton](https://github.com/rzeldent/platformio-espressif32-sunton). This library must reside in the ```<project>/boards``` directory so PlatformIo will automatically recognize these boards.

It is recommended to use the git submodule to include these board definitions automatically.

>[!TIP]
>If you already have a project, clone it with the ```git clone --recurse-submodules```. If creating a new project, use ```git submodule add https://github.com/rzeldent/platformio-espressif32-sunton.git boards``` to add them to your project as a submodule.

### Step 2: Create a new project

Use the standard PlatformIO create project to start a new project. When using a new PlatformIO installation these boards, defined in [platformio-espressif32-sunton](https://github.com/rzeldent/platformio-espressif32-sunton), are not present. Just use a known ESP32 board and correct this later in the platformIO file.

Optionally, you can copy the boards definition to the ```<home>/.platformio\platforms\espressif32\boards``` directory to have them always available but it is probably easier to create the project, add the boards as a git submodule and change the board afterwards. For each supported board there is a board definition.

### Step 3: Add this library to your project

To add this library (and its dependency on LVGL) add the following line to the ```platformio.ini``` file:

From the platformIO registry (version 2.0.0):

```ini
lib_deps = rzeldent/esp32_smartdisplay
```

or the Github repository:

```ini
lib_deps = https://github.com/rzeldent/esp32-smartdisplay.git
```

This will automatically download the library, the LVGL library (as a dependency) and set the defines required for the low level drivers.

### Step 4: Create a settings file for LVGL

LVGL needs a configuration file; ```lv_conf.h```. This file contains information about the fonts, color depths, default background, styles, etc...
The default LVGL template can be found in the LVGL library at the location: ```lvgl/lv_conf_template.h```.
This file must be copied to the include directory and renamed to ```lvgl_conf.h```. Also the ``#if 0`` must be removed to enable the file to be included.
More information about setting up a project with LVGL can be obtained at [LVGL get-started/quick-overview](https://docs.lvgl.io/master/get-started/quick-overview.html#add-lvgl-into-your-project).
I suggest to put the ```lv_conf.h``` file in the include directory and set the build flags to specify the location of the file, see below.

>[!TIP]
>LVGL can also be downloaded manually from: [https://github.com/lvgl/lvgl/archive/master.zip](https://github.com/lvgl/lvgl/archive/master.zip) to extract the template
>or taken from the [appendix](#appendix-lv_confh-example).

Important settings are:

- Only the R5G6B5 format is supported on these panels.

  ```h
  /*Color depth: 1 (1 byte per pixel), 8 (RGB332), 16 (RGB565), 32 (ARGB8888)*/
  #define LV_COLOR_DEPTH 16
  ```

- Because of the SPI interface, the bytes are sent in big endian format so this must be corrected.
  The RGB panel interface takes care of this by swapping the GPIO lines but for the SPI controllers this is not optimal. More information about this [below](#more-on-lv_color_16_swap).

  ```h
  /*Swap the 2 bytes of RGB565 color. Useful if the display has an 8-bit interface (e.g. SPI)*/
  #define LV_COLOR_16_SWAP 1
  ```

- To have a time reference use the milliseconds reference for the Arduino

  ```h
  /*Use a custom tick source that tells the elapsed time in milliseconds and enables this code.*/
  /*It removes the need to manually update the tick with `lv_tick_inc()`)*/
  #define LV_TICK_CUSTOM 1
  ```

- For debugging, enable CPU usage, FPS (Frames per Second) and memory defragmentation

  ```h
  #define LV_USE_PERF_MONITOR 1
  #define LV_USE_MEM_MONITOR 1
  ```

- (Optionally) Include additional fonts.

  ```h
  ...
  #define LV_FONT_MONTSERRAT_22 1
  ...
  ```

- Optionally, only enable widgets that are used to save on code

  ```h
  #define LV_USE_ARC        1
  #define LV_USE_BAR        1
  #define LV_USE_BTN        1
  #define LV_USE_BTNMATRIX  1
  ...
  ```

For debugging it is possible to enable logging from LVGL. The library will output to the debugging output (using ```lv_log_register_print_cb```).
To enable logging, set the define:

```h
/*Enable the log module*/
#define LV_USE_LOG 1
```

By default the logging is only ```LV_LOG_LEVEL_WARN``` but can be adjusted in the ```lv_conf.h```.

More information about the LVGL configuration can be found in the excellent [LVGL documentation](https://docs.lvgl.io/8.3/index.html).

>[!WARNING]
>After the library has been build, changes in the lv_conf.h are no longer applied because libraries are cached.
>To apply these settings, delete the ```.pio``` directory so the libraries will be rebuild.

### Step 5: Copy the build flags below in your project

Especially the definition of the LV_CONF_PATH is critical, this must point to an **absolute path** where the ```lv_conf.h``` file is located. More about this in the [section below](#more-on-lv_confh).

```ini
build_flags =
    -Ofast
    -Wall
    -D CORE_DEBUG_LEVEL=ARDUHAL_LOG_LEVEL_VERBOSE
    # LVGL settings. Point to your lv_conf.h file
    -D LV_CONF_PATH="${PROJECT_DIR}/example/lv_conf.h"
```

The line in the settings logs to the serial console but can be omitted for production builds:

```ini
-D CORE_DEBUG_LEVEL=ARDUHAL_LOG_LEVEL_VERBOSE
```

The -Wall flag can also be removed but outputs all the warnings.

### Step 6: Initialize the display (and touch) in your project

To enable to display in your project call the void ```smartdisplay_init()``` function at startup and set the orientation:

```cpp
void setup()
{
  smartdisplay_init();

  auto disp = lv_disp_get_default();
  // lv_disp_set_rotation(disp, LV_DISP_ROT_90);
  // lv_disp_set_rotation(disp, LV_DISP_ROT_180);
  // lv_disp_set_rotation(disp, LV_DISP_ROT_270);
}
```

and update the timer (and drawing) in the loop:

```cpp
void loop()
{
  lv_timer_handler();
}
```

## Step 7 (Optional): Create your LVGL file or use SquareLine Studio to make a design

There is a good UI designer available for LVGL and free (but some limitations) for personal use: [SquareLine Studio](https://squareline.io/):

[![SquareLine Studio](assets/images/SquareLineStudio.png)](https://squareline.io/)

This tool makes it easy to create transitions, insert images, attach events, work with round screens etc..
A big advantage is that the UI C-code is generated!

## Step 8: Compile, upload and enjoy

These steps should make it possible to run your application on the display!

If there are problems:

- Read this manual again to see if all the steps were followed
- Check if you installed the board definition(s) correctly, [see](#step-2-create-a-new-project)
- Check if the [demo application works](https://github.com/rzeldent/esp32-smartdisplay-demo/tree/feature/esp32s3) and look for differences
- Check if it is a known or previously resolved issue in the [issues](https://github.com/rzeldent/esp32-smartdisplay/issues)
- Refer to the [discussions](https://github.com/rzeldent/esp32-smartdisplay/discussions)
- If all fails, submit an [issue](https://github.com/rzeldent/esp32-smartdisplay/issues), no SLA as this is done in my spare time.

## More on lv_conf.h

To use the LVGL library, a `lv_conf.h` file is required to define the settings for LVGL.
As this file is referenced from the build of LVGL, the **absolute path** must be known.
Normally this file is included in the include directory of your project but also used by the LVGL in the library.

To include it globally, the define must be (for the include directory):

```ini
  -D LV_CONF_PATH=${PROJECT_INCLUDE_DIR}/lv_conf.h
```

>[!TIP]
>The template for the `lv_conf.h` file can be found in the LVGL library at `.pio/libdeps/<board>/lvgl/lv_conf_template.h`.

## More on PSRAM

At boot the ESP32 has approx 315Kb usable memory (from the 520Kb present). Displays with a controller do not have an internal frame buffer to save the state of the display. However, if the display is using the direct 16bits parallel connection, there is no controller and the contents of the RAM is copied to the display at a fast rate. This buffer is allocated from the memory of the ESP32.

If using the normal SRAM, this would (for the ESP32_4827S043 with a 480x270 pixels) snoop away 480x270x2 = 261120 bytes, leaving only 54140 bytes for the application. This can be done and works.
However, the other boards with a resolution of 800x480, 800x480x2 = 768000 bytes are required. This is more than the available RAM.

This is the reason for boards with a direct 16bits parallel connection 8Mb of PSRAM is added and should be used to store this buffer.

The standard definition of the ```esp32-s3-devkitc-1.json``` does not have any configuration for the increases flash size and PSRAM. This is one of the reasons a special board definitions have been created.

## More on LV_COLOR_16_SWAP

The LVGL library has a define called **LV_COLOR_16_SWAP** in the ```lvgl_conf.h```. The value can be 1 or 0.
This variable will swap the byte order of the lv_color16_t. This is required because the SPI is by default LSB first.

Setting this variable to true is recommended for the SPI interfaces: ST7789, GC9A01A, ILI9341 and ST7796. If not, a warning will be issued but the code should work. The parallel 16 bits panels without interface are not affected by this; the GPIO pin layout will change accordingly.

This makes it easier to have only one definition for lv_conf.h and SquareLine.

>[!IMPORTANT]
>If this is not done, the code will run but swapping will be done runtime (and degrading the performance).
>So it is preferable to always set the LV_COLOR_16_SWAP to 1 when using SPI.

Additionally, when using the [SquareLine Studio](https://squareline.io/) for designing the user interface, the display properties (under the project settings) must match this variable.
It needs to be set both in `lv_conf.h` configuration file and the corresponding display properties (16 bit swap) in [SquareLine Studio](https://squareline.io/).

![SquareLine display properties](assets/images/Squareline-display-properties.png)

## LVGL initialization Functions

The library exposes the following functions.

### void smartdisplay_init()

This is the first function that needs to be called.
It initializes the display controller and touch controller and will turn on the display at 50% brightness.

### void smartdisplay_tft_set_backlight(float duty)

Set the brightness of the backlight display.
The range is from [0, 1] so 0 is off, 0.5 is half and 1 is full brightness.

## Rotation of the display and touch

The library supports rotating for most of the controllers using hardware. Support for the direct 16bits parallel connection is done using software emulation (in LVGL).
Rotating the touch is also done when rotating.

Rotating is done calling the ```lv_disp_set_rotation``` function in the LVGL library with the rotation:

```c++
  auto disp = lv_disp_get_default();
  lv_disp_set_rotation(disp, LV_DISP_ROT_90);
```

Some boards are around that have flipped screens, this is probably due to differences during tha manufacturing or using different TFTs. It is possible to correct these boards overriding the default defines.
However if this is encountered a separate board definition is preferable.

## Controlling the RGB led

>[!NOTE]
>Not all boards have a LED. Refer to the [Sunton Boards information](https://github.com/rzeldent/platformio-espressif32-sunton) to see if this is available.

If the board has an RGB led, the define ```BOARD_HAS_RGB_LED``` is defined.
Additionally, the following defines are present for the definition of the GPIO pins:

- LED_R_GPIO
- LED_G_GPIO
- LED_B_GPIO

Before using the RGB LEDs, the GPIOs must be defined as output. This is already done in the [smartdisplay_init()](#void-smartdisplay_init) function.

```c++
  pinmode(LED_R_GPIO, OUTPUT);
  pinmode(LED_G_GPIO, OUTPUT);
  pinmode(LED_B_GPIO, OUTPUT);
```

The LEDs are connected between the GPIO pin and the 3.3V. So the LED will light up if the GPIO is set to LOW (inverted).
![LEDS schematic](assets/images//LEDS.png)

For example: set the RGB led to red is done by the following code:

```c++
  digitalWrite(LED_R_GPIO, false);
  digitalWrite(LED_G_GPIO, true);
  digitalWrite(LED_B_GPIO, true);
```

To have more colors than the 8 RGB combinations, PWM can be used to mix the colors.
To do this, attach a PWM channel to each GPIO pin to modulate the intensity.

>[!WARNING]
>The number of PWM channels is limited and some channels are reserved for some functions
>e.g. channel 0 is used by the tone() function but can be used if this function is not called.

Example:
Set PWM channel 0 to 5000Hz with a resolution of 8 bits (256 levels) and attach it to the red LED.
Next, set the level to 25%. This is 192 (256 - 25%) because of the inverted output.

>[!CAUTION]
>The functions for using PWM will change in the upcoming release of the IDF and will break the usage of the PWM functions!

ESP_ARDUINO_VERSION_MAJOR < 3:

```c++
  ledcSetup(0, 5000, 8);
  ledcAttachPin(LED_R_GPIO, 0);
  ledcWrite(0, 192);
```

ESP_ARDUINO_VERSION_MAJOR >= 3

```c++
  ledcAttach(LED_R_GPIO, 0, 8);
  ledcWrite(LED_R_GPIO, 192);
```

## Reading the CDS (light sensor)

>[!NOTE]
>Not all boards have a light sensor. Refer to the [Sunton Boards information](https://github.com/rzeldent/platformio-espressif32-sunton) to see if this is available.

If the board has a CDS photo resistor (Cadmium Sulfide, CdS) light sensor, the define ```BOARD_HAS_CDS``` is defined.
It is attached to the analogue input of the ESP32 with two resistors between the GND and the VCC. When the CDS is covered, it's resistance is in the order of mega&Omega; but in bright light can drop to a few 100&Omega;.

![CDS](assets/images/CDS.png)

To use the sensor, the define ```CDS_GPIO``` indicates the analogue port to read.
Setting the port to input and the attenuation is already done in the [smartdisplay_init()](#void-smartdisplay_init) function.

```c++
  analogSetAttenuation(ADC_0db); // 0dB(1.0x) 0~800mV
  pinMode(CDS_GPIO, INPUT);
```

Next, read the value using:

```c++
  auto value = analogReadMilliVolts(CDS_GPIO);
```

The value ranges from approximately 75mV (not covered) to 400mV (completely covered).

## Controlling the speaker

>[!NOTE]
>Not all boards have a LED. Refer to the [Sunton Boards information](https://github.com/rzeldent/platformio-espressif32-sunton) to see if this is available.

An 8&Omega; speaker can be connected to the output marked SPEAK. This is a 1.25 JST connector.
Setting the speaker GPIO to output is already done in the [smartdisplay_init()](#void-smartdisplay_init) function.

```c++
pinmode(SPEAK_GPIO, OUTPUT)
```

Beeps can be generated by generating a PWM signal on the SPEAK_GPIO or using the tone function:

```c++
// Uses PWM Channel 0
tone(SPEAK_GPIO, frequency, duration);
```

To produce "real" audio connect the internal 8 bits D2A converter in the ESP32. Because the speaker is connected to GPIO26, this is the DAC2 (Left Channel).

>[!TIP]
>Make sure the I2S connection is only to the LEFT channel. GPIO25, the right channel, is connected on some boards to the GT911 touch controller and creates strange results.

The audio is a bit distorted. [HexeguitarDIY](https://github.com/hexeguitar/ESP32_LCD_PIO) has a fix for that by changing the resistor values to prevent distortion.
[![HexeguitarDIY Audio mod](https://img.youtube.com/vi/6JCLHIXXVus/0.jpg)](https://www.youtube.com/watch?v=6JCLHIXXVus)

## Appendix: Template to support ALL the boards

The platformio.ini file below supports all the boards. This is useful when running your application on multiple boards. If using one board only, uncomment the ```default_envs ``` for that board in the ```[platformio]``` section.

>[!TIP]
>When building using a pipeline (github action), the ini below, with all the boards, can be used automatically create builds for all the boards.

```ini
[platformio]
#default_envs = esp32-1732S019C
#default_envs = esp32-1732S019N
#default_envs = esp32-2424S012C
#default_envs = esp32-2424S012N
#default_envs = esp32-2432S024C
#default_envs = esp32-2432S024N
#default_envs = esp32-2432S024R
#default_envs = esp32-2432S028R
#default_envs = esp32-2432S028Rv2
#default_envs = esp32-2432S032C
#default_envs = esp32-2432S032N
#default_envs = esp32-2432S032R
#default_envs = esp32-3248S035C
#default_envs = esp32-3248S035R
#default_envs = esp32-4827S043C
#default_envs = esp32-4827S043N
#default_envs = esp32-4827S043R
#default_envs = esp32-4848S040C
#default_envs = esp32-8048S043C
#default_envs = esp32-8048S043N
#default_envs = esp32-8048S043R
#default_envs = esp32-8048S050C
#default_envs = esp32-8048S050N
#default_envs = esp32-8048S050R
#default_envs = esp32-8048S070C
#default_envs = esp32-8048S070N

[env]
platform = espressif32
framework = arduino

monitor_speed = 115200
monitor_rts = 0
monitor_dtr = 0
monitor_filters = esp32_exception_decoder

# Partition scheme for OTA
board_build.partitions = min_spiffs.csv

build_flags =
    -Ofast
    #-Wall
    -D CORE_DEBUG_LEVEL=ARDUHAL_LOG_LEVEL_VERBOSE
    # LVGL settings
    #-DLV_CONF_PATH=${PROJECT_INCLUDE_DIR}/lv_conf.h
    
lib_deps =
    # From library.json. Normally not needed!
    lvgl/lvgl

[env:esp32-1732S019C]
board = esp32-1732S019C

[env:esp32-1732S019N]
board = esp32-1732S019N

[env:esp32-2424S012C]
board = esp32-2424S012C

[env:esp32-2424S012N]
board = esp32-2424S012N

[env:esp32-2432S024C]
board = esp32-2432S024C

[env:esp32-2432S024N]
board = esp32-2432S024N

[env:esp32-2432S024R]
board = esp32-2432S024R

[env:esp32-2432S028R]
board = esp32-2432S028R

[env:esp32-2432S028Rv2]
board = esp32-2432S028Rv2

[env:esp32-2432S032C]
board = esp32-2432S032C

[env:esp32-2432S032N]
board = esp32-2432S032N

[env:esp32-2432S032R]
board = esp32-2432S032R

[env:esp32-3248S035C]
board = esp32-3248S035C

[env:esp32-3248S035R]
board = esp32-3248S035R

[env:esp32-4827S043C]
board = esp32-4827S043C

[env:esp32-4827S043N]
board = esp32-4827S043N

[env:esp32-4827S043R]
board = esp32-4827S043R

[env:esp32-4848S040C]
board = esp32-8048S043C

[env:esp32-8048S043C]
board = esp32-8048S043C

[env:esp32-8048S043N]
board = esp32-8048S043N

[env:esp32-8048S043R]
board = esp32-8048S043R

[env:esp32-8048S050C]
board = esp32-8048S050C

[env:esp32-8048S050N]
board = esp32-8048S050N

[env:esp32-8048S050R]
board = esp32-8048S050R

[env:esp32-8048S070C]
board = esp32-8048S070C

[env:esp32-8048S070N]
board = esp32-8048S070N
```

## Appendix: External dependencies

The following libraries are used from the [EspressIf component registry](https://components.espressif.com/):

| Name                  | Version |
|---                    |---      |
| [ESP LCD CG9A01](https://components.espressif.com/api/download/?object_type=component&object_id=6f06ecdf-97a6-4eea-ad4f-c00d11bd970a)         | v1.2    |
| [ESP LCD ILI9341](https://components.espressif.com/api/download/?object_type=component&object_id=680fe7b6-c70b-4560-acf9-919e5b8fa192)        | v2.0    |
| [ESP LCD ST7796](https://components.espressif.com/api/download/?object_type=component&object_id=eb6095d1-642a-4e14-9daf-d46db8a1f354)         | v1.2.1  |
| [ESP LCD Touch](https://components.espressif.com/api/download/?object_type=component&object_id=bb4a4d94-2827-4695-84d1-1b53383b8001)          | v1.1.1  |
| [ESP LCD Touch CST816S](https://components.espressif.com/api/download/?object_type=component&object_id=cc8ef108-15e8-48cf-9be8-3c7e89ca493e)  | v1.0.3  |
| [ESP LCD Touch GT911](https://components.espressif.com/api/download/?object_type=component&object_id=4f44d570-8a04-466e-b4bb-429f1df7a9a1)    | v1.1.0  |
| [ESP LCD Touch Driver](https://components.espressif.com/api/download/?object_type=component&object_id=225971c2-051f-4619-9f91-0080315ee8b8)   | v1.2.0  |

## Version history

- December 2023
  - 2.0.2 release
  - Updated documentation
  - Added rotation
  - Added ESP32_1732S019N/C
- November 2023
  - Major version update: 2.0.0
  - Rewrite of the library to support the new ESP32-C3 and ESP32-S3 panels
  - Use the new Espressif esp_lcd interface
  - Use C instead of cpp
- October 2023
  - Added support for esp32_8048S034N/C
  - Added option for flipped/mirrored TFT's
  - Changed default RGB order to BGR
  - Version 1.0.8 and 1.0.9
- September 2023
  - Added support for ESP32_2432S024N/R/S
  - Version 1.0.7
- August 2023
  - Added support for esp32_8048S070N/C
  - Display buffer size configurable
- February 2023
  - Version 1.0.3
  - Added variable for the LCD panel RGB/BGR order
- February 2023
  - Version 1.0.3
  - Added mutex for access to lvgl, required for multithreading
  - Changed RGB led input to lv_color32_t
- December 2022
  - Initial version 1.0.2.
  - Drivers for ESP32_2432S028R, ESP32_3248S035R and ESP32_3248S035C displays working.
  - Sound output
  - RGB Led output
  - CDS light sensor input
- July 2022
  - Initial work started
