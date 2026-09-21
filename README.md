# ESP32-S3 AI Assistant Robot

ESP-IDF firmware for an ESP32-S3 voice assistant robot with audio input/output, an OLED display, status LED, buttons, motor control, Wi-Fi, OTA support, and MCP tools.

The application is based on the Xiaozhi Assistant firmware and is configured for the AI Assistant Robot hardware profile in `main/boards/AI-Assistant-Robot`.

## Features

- ESP32-S3 voice assistant application
- Microphone and speaker audio over I2S
- 128x32 or 128x64 SSD1306 OLED support
- Wi-Fi configuration and network operation
- OTA update support
- LVGL-based display support and language assets
- MCP server tools, including robot and lamp control hooks
- Dual OTA application partitions
- Generated voice, font, model, and display assets stored in a dedicated SPIFFS partition
- Motor control for two N20 motors through a TC1508 module

## Hardware

The robot profile uses an ESP32-S3 with 16 MB flash and octal PSRAM. Confirm that the physical board has the same wiring before powering motors or flashing firmware.

### AI Assistant Robot pin map

| Function | GPIO |
| --- | ---: |
| I2S microphone WS | 4 |
| I2S microphone SCK | 5 |
| I2S microphone data | 6 |
| I2S speaker data out | 7 |
| I2S speaker BCLK | 15 |
| I2S speaker LRCLK | 16 |
| Built-in LED | 48 |
| Boot button | 0 |
| Touch button | 47 |
| Volume up | 40 |
| Volume down | 39 |
| OLED SDA | 41 |
| OLED SCL | 42 |
| Lamp/MCP test output | 18 |
| Left motor forward | 12 |
| Left motor backward | 13 |
| Right motor forward | 14 |
| Right motor backward | 21 |

The default audio mode is simplex I2S. The audio sample rates are 16 kHz input and 24 kHz output. To use duplex wiring, remove `AUDIO_I2S_METHOD_SIMPLEX` and verify the alternate pin definitions in `main/boards/AI-Assistant-Robot/config.h`.

## Requirements

- ESP32-S3 board matching the selected hardware profile
- USB data cable and a suitable USB/UART driver
- ESP-IDF 5.4.0 or newer
- Python supplied by the ESP-IDF installation
- CMake and Ninja supplied by ESP-IDF
- A serial port available for flashing and monitoring
- Enough power for the ESP32-S3, display, audio hardware, and motors

The repository's VS Code ESP-IDF configuration currently points to `D:\esp\v5.5.5\esp-idf` and sets the target to `esp32s3`. Update `.vscode/settings.json` if ESP-IDF is installed elsewhere.

## Get the project ready

Open an ESP-IDF PowerShell or Command Prompt, then change to the project directory:

```powershell
cd D:\Projects\ESP32S3-Robot-main
```

If the ESP-IDF environment is not active, export it using the ESP-IDF installation for your system. For the current installation, the usual PowerShell command is:

```powershell
D:\esp\v5.5.5\export.ps1
```

Check that the tools are available:

```powershell
idf.py --version
python --version
```

The project component manifest requires IDF `>=5.4.0`. The first configure/build may download managed components, so an internet connection may be required.

## Select the target and board

Always select the target before the first build or after changing targets:

```powershell
idf.py set-target esp32s3
```

Open project configuration:

```powershell
idf.py menuconfig
```

Under **Xiaozhi Assistant**, verify:

1. The board is **AI-Assistant-Robot** (`BOARD_TYPE_QEBABE_XIAOCHE`).
2. The OLED size matches the hardware: 128x32 or 128x64.
3. The desired default language is selected.
4. Asset flashing is enabled unless assets are already present on the device.
5. The flash size is 16 MB.
6. The custom partition table is `partitions/v2/16m.csv`.

The checked-in defaults select `BOARD_TYPE_QEBABE_XIAOCHE` and the 128x64 OLED. The board profile in `main/boards/AI-Assistant-Robot/config.h` contains the robot GPIO map.

After changing menuconfig, save the configuration and reconfigure the project:

```powershell
idf.py reconfigure
```

## Build

For a normal build:

```powershell
idf.py build
```

For a clean rebuild after changing the target, partition table, board, or flash size:

```powershell
idf.py fullclean
idf.py set-target esp32s3
idf.py build
```

The build produces the firmware files in `build/`, including the bootloader, partition table, application, and generated asset image.

## Flash and monitor

Find the serial port assigned to the board in Windows Device Manager. Replace `COM7` below with the actual port.

Flash the complete project and open the serial monitor:

```powershell
idf.py -p COM7 flash monitor
```

The normal flash operation includes:

- Bootloader
- Partition table
- OTA application image
- Generated `assets` image at offset `0x800000`

To flash without opening the monitor:

```powershell
idf.py -p COM7 flash
```

To monitor an already flashed device:

```powershell
idf.py -p COM7 monitor
```

Exit the ESP-IDF monitor with `Ctrl+]`.

If the board does not enter download mode automatically, hold the BOOT button, press and release RESET/EN, then release BOOT when flashing begins. The exact button labels vary by board.

## Flash layout

The default partition table is `partitions/v2/16m.csv`:

| Partition | Type | Offset | Size |
| --- | --- | ---: | ---: |
| `nvs` | data/nvs | `0x9000` | 16 KB |
| `otadata` | data/ota | `0xD000` | 8 KB |
| `phy_init` | data/phy | `0xF000` | 4 KB |
| `ota_0` | app/ota_0 | `0x20000` | 3.9375 MB |
| `ota_1` | app/ota_1 | after `ota_0` | 3.9375 MB |
| `assets` | data/spiffs | `0x800000` | 8 MB |

The firmware expects the partition label `assets`. Do not replace the partition table with a 2 MB or 4 MB layout when using the 16 MB defaults.

## Assets

With the default `FLASH_DEFAULT_ASSETS` setting, CMake generates `build/generated_assets.bin` from the selected language, fonts, emoji, and speech-recognition models. The image is automatically included in `idf.py flash` and written to the `assets` partition.

Asset options are available in `idf.py menuconfig`:

- **Flash Default Assets**: generate and flash the normal project assets.
- **Flash Custom Assets**: flash the file configured by `CUSTOM_ASSETS_FILE`.
- **Flash Emote Assets**: generate expression/emote assets when that mode is enabled.
- **Do not flash assets**: leave the existing asset partition unchanged.

If the asset build fails, check that the managed components and model/font directories exist, then run:

```powershell
idf.py reconfigure
idf.py build
```

## First boot

1. Connect the board by USB and flash the firmware.
2. Open the serial monitor and watch startup logs.
3. Complete the device's Wi-Fi configuration flow.
4. Confirm that the display, microphone, speaker, buttons, LED, and motors are wired according to the selected board profile.
5. Keep the robot's wheels lifted during the first motor test.

The application initializes NVS for Wi-Fi and settings. If NVS is corrupt or incompatible, the firmware erases and recreates it during startup.

## VS Code workflow

Install the Espressif ESP-IDF extension and open this repository as the workspace. Ensure the extension uses the ESP-IDF installation configured in `.vscode/settings.json`.

Useful commands from the ESP-IDF command palette:

- **ESP-IDF: Set Espressif Device Target**: choose `esp32s3`.
- **ESP-IDF: SDK Configuration Editor**: edit `sdkconfig`.
- **ESP-IDF: Build Your Project**: compile the firmware.
- **ESP-IDF: Flash Your Project**: upload firmware to the selected serial port.
- **ESP-IDF: Monitor Your Device**: view serial logs.

The command-line workflow is usually easier to reproduce and is documented above.

## Troubleshooting

### Flash size mismatch

A message such as `Partition table ... larger than flash size` means the active `sdkconfig` has the wrong flash size. Fix it by regenerating the target configuration:

```powershell
idf.py fullclean
idf.py set-target esp32s3
idf.py reconfigure
idf.py build
```

Then verify that `CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y` is present in `sdkconfig`.

### Wrong target

If the build mentions ESP32 instead of ESP32-S3, reset the target:

```powershell
idf.py set-target esp32s3
```

Do not reuse an old `sdkconfig` generated for another ESP32 family without reconfiguring it.

### Port not found or upload fails

- Use a USB data cable, not a charge-only cable.
- Close serial monitors and other applications using the COM port.
- Confirm the port in Device Manager.
- Hold BOOT while resetting the board to force download mode.
- Try a lower USB hub load and power the motor hardware separately if necessary.
- Use `idf.py -p COMx flash` with the correct port.

### Board works but display or audio does not

Confirm that the selected board and OLED size match the physical hardware. Then compare the wiring with `main/boards/AI-Assistant-Robot/config.h`. Incorrect I2S wiring can produce silence or noise even when the firmware builds successfully.

### Motors move unexpectedly

Disconnect motor power during development and keep the wheels off the table during tests. Verify the TC1508 wiring and all four motor GPIO assignments before enabling motor control.

### Stale generated files

If configuration changes appear to have no effect, use:

```powershell
idf.py fullclean
idf.py set-target esp32s3
idf.py build
```

Do not delete or edit generated files inside `build/` by hand while a build is running.

## Project layout

```text
.
|- main/                          Application source and board profiles
|  |- boards/AI-Assistant-Robot/  Robot configuration and GPIO map
|  |- audio/                      Audio codecs and audio service
|  |- display/                    OLED/LVGL display code
|  |- protocols/                  Network protocols
|  |- web_server/                 Device web server
|  |- assets/                     Languages and source assets
|  |- CMakeLists.txt              Component and asset build rules
|  `- Kconfig.projbuild           Project configuration options
|- partitions/v2/16m.csv          16 MB OTA and assets partition table
|- scripts/                       Asset and language generation scripts
|- sdkconfig.defaults             Common project defaults
|- sdkconfig.defaults.esp32s3    ESP32-S3 flash, PSRAM, and performance defaults
|- dependencies.lock              Managed component versions
`- CMakeLists.txt                 Top-level ESP-IDF project file
```

## License

See [LICENSE](LICENSE) for the project license.
