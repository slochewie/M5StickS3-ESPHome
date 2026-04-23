# M5StickS3 ESPHome Config for Home Assistant Counter + Dual Button Unit

This repository contains an ESPHome configuration for an **M5Stack M5StickS3** connected to an **M5Stack Dual Button Unit**. The device acts as a small Wi‑Fi counter controller and display for a Home Assistant counter entity.

The current YAML:

- boots the M5StickS3 with Arduino framework and PSRAM enabled
- initializes the StickS3 PMIC and power rails at boot
- enables the LCD backlight and updates the display
- shows the current Home Assistant counter value on screen
- uses the Dual Button Unit to increment and decrement the counter
- uses onboard **Key2** to reset the counter
- exposes status, uptime, Wi‑Fi RSSI, and a local web server

The behavior described below is based on the current YAML file. fileciteturn3file0

## What the YAML currently does

### 1. Board and framework setup

The config targets `esp32-s3-devkitc-1` and uses the **Arduino** framework. It also enables **octal PSRAM at 80 MHz**. This gives the display and libraries enough memory headroom for the StickS3 setup. fileciteturn3file0

### 2. PMIC and power initialization on boot

On boot, the config pulls in `Wire.h` and `M5PM1.h`, then initializes the PMIC over I2C on **GPIO47 (SDA)** and **GPIO48 (SCL)**. It enables:

- DCDC power
- LDO power
- charging
- external 5V boost
- the **L3B path** used for the LCD/audio rail

This is done inside the `on_boot` lambda before the display is updated. The PMIC init sequence and the follow-up backlight/display update are explicitly defined in the YAML. fileciteturn3file0

### 3. LCD backlight control

The LCD backlight is driven from **GPIO38** using an LEDC output at **1000 Hz**. That output is wrapped in a monochromatic light entity named **StickS3 Backlight**.

At boot, the config turns the backlight on to **15% brightness**, waits 500 ms, then forces a display refresh. fileciteturn3file0

### 4. External custom display component

Instead of using ESPHome's stock `st7789v`, `ili9xxx`, or `mipi_spi` display platforms, this config uses a **local external component**:

```yaml
external_components:
  - source:
      type: local
      path: components
```

and the display itself is declared as:

```yaml
display:
  - platform: m5sticks3_display
```

That means this repo depends on a custom display driver in the local `components/` directory. The YAML refreshes that display every **500 ms**. fileciteturn3file0

### 5. What is shown on the screen

The display code draws:

- a black background
- a title: **Capacity Counter A**
- the current value of the Home Assistant counter in large yellow text
- `--` in orange if the counter value is unavailable (`NaN`)

This is rendered from the Home Assistant sensor `counter.capacity_counter`. fileciteturn3file0

### 6. Home Assistant counter integration

The YAML imports the Home Assistant entity:

```yaml
counter.capacity_counter
```

as an internal ESPHome sensor named `capacity_counter_value`.

That value is used only inside the device for display rendering; it is marked `internal: true`, so it is not separately exposed as a user-facing sensor entity by ESPHome. fileciteturn3file0

### 7. Button behavior

#### Onboard buttons

- **Key1 (GPIO11)** is currently configured as a normal GPIO binary sensor with pull-up and inverted logic.
- **Key2 (GPIO12)** is also configured as a GPIO binary sensor with pull-up and inverted logic.

At the moment:

- **Key1** has no action attached
- **Key2** resets the Home Assistant counter when pressed

The reset action is sent using:

```yaml
homeassistant.action:
  action: counter.reset
```

for `counter.capacity_counter`. fileciteturn3file0

#### Dual Button Unit

The attached Dual Button Unit is wired as:

- **Red button** on **GPIO9**
- **Blue button** on **GPIO10**

Both buttons are inverted and debounced with 10 ms delayed on/off filters.

Their actions are:

- **Red button** → decrement `counter.capacity_counter`
- **Blue button** → increment `counter.capacity_counter`

These actions are sent to Home Assistant using `homeassistant.action` calls. fileciteturn3file0

## Services and connectivity

The YAML enables:

- **Wi‑Fi** using secrets
- **ESPHome API** with encryption
- **OTA** updates
- **captive portal** fallback
- a **web server** on port 80
- **debug logging** at `DEBUG` level

This lets the device work both as a Home Assistant node and as a directly viewable ESPHome web UI device. fileciteturn3file0

## Entities created by this config

The YAML currently creates or uses these main entities:

### Display / light

- `StickS3 Backlight`

### Binary sensors

- `StickS3 Status`
- `StickS3 Key1`
- `StickS3 Key2`
- `Dual Button Red`
- `Dual Button Blue`

### Sensors

- `StickS3 Uptime`
- `StickS3 WiFi RSSI`
- internal sensor for `counter.capacity_counter`

### Home Assistant actions triggered

- `counter.increment`
- `counter.decrement`
- `counter.reset`

All of these are directly present in the current YAML. fileciteturn3file0

## Required dependencies

This config depends on the following libraries declared in YAML:

- `Wire`
- `M5PM1`
- `M5Unified`
- `M5GFX`

It also depends on a local ESPHome external component under `components/`, because the display platform is `m5sticks3_display`. Without that local component, the display portion will not compile. fileciteturn3file0

## Secrets expected

The YAML expects these secrets to exist:

- `api_encryption_key`
- `ota_password`
- `wifi_ssid`
- `wifi_password`
- `ap_password`

These are referenced directly in the config. fileciteturn3file0

## Current workflow summary

In plain English, the device currently works like this:

1. Power on the M5StickS3
2. Initialize the PMIC and LCD-related power rail
3. Turn on the backlight
4. Connect to Wi‑Fi and Home Assistant
5. Read the current `counter.capacity_counter` value
6. Show that value on the screen
7. Let the Dual Button Unit increment and decrement the counter
8. Let onboard Key2 reset the counter

That is the full functional behavior implemented in the current YAML. fileciteturn3file0

## Notes

- The display implementation is **custom**, not stock ESPHome.
- The PMIC setup is essential for this board.
- Key1 is currently configured as an input only; it does not yet trigger any action in this YAML.

## License / usage

Adapt freely for your own M5StickS3 projects. If you publish this repo, it is a good idea to also include:

- the local `components/` folder
- a sample `secrets.yaml`
- wiring notes for the M5Stack Dual Button Unit
