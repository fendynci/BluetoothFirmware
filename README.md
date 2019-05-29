# Bluetooth Firmware 1.0.0 #

The Bluetooth Firmware library contains the firmware strings needed to [initialize an imp API **bluetooth** object](https://developer.electricimp.com/api/hardware/bluetooth/open). The library contains the latest recommended firmware for use with the imp004m and imp006 modules. The imp004m firmware is approximately 16KB in size. The imp006 firmware is approximately 60KB in size. The firmware will only consume memory if referenced in your code.

**To include this library in your project, add** `#require "bt_firmware.lib.nut:1.0.0"` **at the top of your code**

## Library Usage ##

The library consists of an enum, *BT_FIRMWARE*, with the following elements, all strings.

| Element | Value |
| --- | --- |
| *VERSION* | Bluetooth Firmware library release version |
| *CYW_43438* | Firmware for the imp004m |
| *CYW_43455* | Firmware for the imp006, WiFi/Bluetooth click |

### imp004m Example ###

```squirrel
bt <- hardware.bluetooth.open(bt_uart, BT_FIRMWARE.CYW_43438);
```

### imp006 Example ###

```squirrel
bt <- hardware.bluetooth.open(bt_uart, BT_FIRMWARE.CYW_43455);
```

## Full Examples ##

This library can be used on either the agent or the device. These examples will walk through basic usage for both.

### Library On Device ###

When the library is used on the device please be sure your application has enough memory to store the firmware.

#### Device Code ####

```squirrel
#require "bt_firmware.lib.nut:1.0.0"

// Set up BT on imp004m Breakout Board
bt <- null;
bt_uart <- hardware.uartFGJH;
bt_lpo_in <- hardware.pinE;
bt_reg_on <- hardware.pinJ;

// Boot up BT
bt_lpo_in.configure(DIGITAL_OUT, 0);
bt_reg_on.configure(DIGITAL_OUT, 1);

// Pause to ensure pins are in the newly
// configured state before proceeding...
imp.sleep(0.05);

local start = hardware.millis();

try {
    // Instantiate BT with the imp004m firmware
    bt = hardware.bluetooth.open(bt_uart, BT_FIRMWARE.CYW_43438);
    server.log("BLE initialized after " + (hardware.millis() - start) + " ms");
} catch (err) {
    server.error(err);
    server.log("BLE failed after " + (hardware.millis() - start) + " ms");
}
```

### Library On Agent ###

When the library is used on the agent, the firmware will need to be sent to the device, so that the device will be able to boot Bluetooth. The device can persist the received firmware in its SPI flash storage.

#### Agent Code ####

```squirrel
#require "bt_firmware.lib.nut:1.0.0"

device.on("get.firmware", function(ignore) {
    // Send imp004m BT firmware
    server.log("Sending device bluetooth firmware");
    device.send("set.firmware", BT_FIRMWARE.CYW_43438);
})
```

#### Device Code ####

```squirrel
#require "SPIFlashFileSystem.device.lib.nut:2.0.0"

const BT_FIRMWARE_FILE_NAME = "btFirmware";

// Store Bluetooth firmware in SPI Flash
function storeBTFirmware(firmware) {
    server.log("Storing bluetooth firmware to SPI flash");

    // We only want one version of firmware stored at a time, so
    // erase if we already have stored firmware
    if (haveStoredFirmware) sffs.eraseFile(BT_FIRMWARE_FILE_NAME);

    local file = sffs.open(BT_FIRMWARE_FILE_NAME, "w");
    file.write(firmware);
    file.close();
}

// Retrieve Bluetooth firmware from SPI Flash
function getBTFirmware() {
    server.log("Retrieving stored bluetooth firmware");
    if (!haveStoredFirmware) return null;

    local file = sffs.open(BT_FIRMWARE_FILE_NAME, "r");
    local firmware = file.read();
    file.close();
    return firmware;
}

// Use stored firmware to boot bluetooth
function bootBT() {
    server.log("Booting bluetooth using firmware stored in SPI flash");
    bt_lpo_in.configure(DIGITAL_OUT, 0);
    bt_reg_on.configure(DIGITAL_OUT, 1);

    // Pause to ensure pins are in the newly
    // configured state before proceeding...
    imp.sleep(0.05);

    local start = hardware.millis();

    try {
        // Instantiate BT
        bt = hardware.bluetooth.open(bt_uart, getBTFirmware());
        server.log("BLE initialized after " + (hardware.millis() - start) + " ms");
    } catch (err) {
        server.error(err);
        server.log("BLE failed after " + (hardware.millis() - start) + " ms");
    }
}

// Configure global imp004m bluetooth hardware variables
bt <- null;
bt_uart <- hardware.uartFGJH;
bt_lpo_in <- hardware.pinE;
bt_reg_on <- hardware.pinJ;

// Configure SPI flash storage
sffs <- SPIFlashFileSystem(0x000000, 0x020000);
sffs.init();
haveStoredFirmware <- sffs.fileExists(BT_FIRMWARE_FILE_NAME);

// Create listener for firmware message from agent
agent.on("set.firmware", function(firmware) {
    server.log("Received bluetooth firmware from agent.");
    storeBTFirmware(firmware);
    bootBT();
});

if (haveStoredFirmware) {
    // Boot bluetooth with stored firmware
    server.log("We have stored bluetooth firmware");
    bootBT();
} else {
    // Get bluetooth firmware from agent
    server.log("No stored bluetooth firmware. Requesting firmware from agent...");
    agent.send("get.firmware", null);
}
```

## License ##

This library is licensed under the [MIT License](./LICENSE).<br />Firmware copyright Cypress Semiconductor Corp.
