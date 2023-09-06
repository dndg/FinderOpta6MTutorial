---
title: 'Getting Started with Opta and Finder 6M'
description: "Learn how to read 6M registers using the Modbus protocol on Opta."
author: 'Fabrizio Trovato'
libraries:
  - name: 'ArduinoRS485'
    url: https://www.arduino.cc/reference/en/libraries/arduinors485
  - name: 'ArduinoModbus'
    url: https://www.arduino.cc/reference/en/libraries/arduinomodbus
difficulty: intermediate
tags:
  - Getting-started
  - ModbusRTU
  - RS-485
  - Finder 6M
software:
  - ide-v1
  - ide-v2
  - arduino-cli
  - web-editor
hardware:
  - hardware/07.opta/opta-family/opta
---

## Overview

Among the protocols supported by the Opta, we find Modbus RTU. In this tutorial
we will learn how to implement Modbus RTU communication over RS-485 between the
Opta and a Finder 6M power analyzer. In particular, we are going to learn how
to use the Opta to configure a Finder 6M and read its registers.

## Goals

* Learn how to establish RS-485 interface connectivity between the Opta and a
  Finder 6M device.
* Learn how to use the Modbus RTU communication protocol to configure and read
  the registers of a Finder 6M device.

## Required Hardware and Software

### Hardware Requirements

* Opta PLC with RS-485 support (x1).
* Finder 6M power analyzer (x1).
* 12VDC/500mA DIN rail power supply (x1).
* USB-C® cable (x1).
* Wire with either specification for RS-485 connectivity (x2):
  * STP/UTP 24-18AWG (Unterminated) 100-130Ω rated.
  * STP/UTP 22-16AWG (Terminated) 100-130Ω rated.

### Software Requirements

* [Arduino IDE 1.8.10+](https://www.arduino.cc/en/software), [Arduino IDE
2.0+](https://www.arduino.cc/en/software) or [Arduino Web
Editor](https://create.arduino.cc/editor).
* If you choose an offline Arduino IDE, you must install the `ArduinoRS485` and
`ArduinoModbus` libraries. You can install them using the Library Manager of
the Arduino IDE.
* [Example code](assets/Opta6MExample.zip).

## Finder 6M and the Modbus RTU Protocol

Finder 6M power analyzers provide access to a series of *holding registers* via
the Modbus RTU protocol over RS-485 serial connection.

As documented in the [6M Modbus communication protocol
document](https://cdn.findernet.com/app/uploads/Modbus_RS485_6MTx.pdf),
measures on Finder 6M devices are available on Modbus as a series of 16-bit
reads: for example, the Energy measurement is available as a 32-bit value
obtained by combining the read of the two adjacent 16-bit registers located at
Modbus addresses `40089` and `40090`. Note that, for Finder 6M devices all
offsets are *register* offsets, not *byte* offsets. Additionally, on Finder 6M
devices Modbus addressing starts from `0`: this means that for example ModBus
address `40006` must be accessed as *holding register* number `5`.

For more insights on the Modbus communication protocol, take a look at this
[Modbus article](https://docs.arduino.cc/learn/communication/modbus): all the
functionalities provided by the `ArduinoModbus` library are supported by the
Opta.

## Instructions

### Setting Up the Arduino IDE

This tutorial will need [the latest version of the Arduino
IDE](https://www.arduino.cc/en/software). If it is your first time setting up
the Opta, check out the [getting started
tutorial](/tutorials/opta/getting-started).

Make sure you install the latest version of the
[ArduinoModbus](https://www.arduino.cc/reference/en/libraries/arduinomodbus/)
and the
[ArduinoRS485](https://www.arduino.cc/reference/en/libraries/arduinors485/)
libraries, as they will be used to implement the Modbus RTU communication
protocol.

### Connecting the Opta and Finder 6M

To observe actual measurements we will need to connect the Finder 6M power
analyzer to the power grid and provide an adequate load. We will also need to
power the Opta with the 12VDC/500mA supply and to correctly setup the RS-485
serial connection: the diagram below shows the correct wiring setup between the
two devices.

![Connecting Opta and Finder 6M](assets/connection.svg)

For the example code to work, we will need to set the Finder 6M communication
parameters as follows:

* Modbus address `1`.
* Baudrate `38400`.

We can achieve this setting `UP` both DIP switches on the 6M, as explained on
page 6 of the [User
Guide](https://cdn.findernet.com/app/uploads/6M.Tx-User-Guide.pdf).

### Code Overview

The goal of the following example is to configure the Finder 6M power analyzer
using the Opta, and then to read some of the measurements in the registers of
the 6M and print them to the serial console.

The full code of the example is available [here](assets/Opta6MExample.zip):
after extracting the files the sketch can be compiled and uploaded to the Opta.

#### Configuring the Finder 6M

In the `setup()` we are going to:

* Configure the Modbus parameters according to [the Modbus over serial line
  guide](https://modbus.org/docs/Modbus_over_serial_line_V1_02.pdf).
* Set a custom Modbus address and Baudrate for our 6M.

```cpp
#include <ArduinoModbus.h>
#include <ArduinoRS485.h>
#include "finder-6m.h"

constexpr uint8_t MODBUS_6M_DEFAULT_ADDRESS = 1;
constexpr uint8_t MODBUS_6M_ADDRESS = 3;

void setup()
{
    Serial.begin(BAUDRATE);

    RS485.setDelays(PREDELAY, POSTDELAY);
    ModbusRTUClient.setTimeout(TIMEOUT);
    if (ModbusRTUClient.begin(BAUDRATE, SERIAL_8N1) != 1)
    {
        while (1)
            ;
    }

    // Change Modbus address
    modbus6MWrite16(MODBUS_6M_DEFAULT_ADDRESS, FINDER_6M_REG_MODBUS_ADDRESS, MODBUS_6M_ADDRESS);
    // Baudrate 38400 has code 5
    modbus6MWrite16(MODBUS_6M_DEFAULT_ADDRESS, FINDER_6M_REG_BAUDRATE, FINDER_6M_BAUDRATE_CODE_38400);
    // Save above settings
    if (modbus6MWrite16(MODBUS_6M_DEFAULT_ADDRESS, FINDER_6M_REG_COMMAND, FINDER_6M_COMMAND_SAVE))
    {
        // We have 30 seconds to lower the DIP switches
        Serial.println("Waiting 30s while you:");
        Serial.println("1. Power OFF the 6M.");
        Serial.println("2. Set both DIP switches DOWN.");
        Serial.println("3. Power back ON the 6M.");
        delay(30000);
    }
    else
    {
        while (1)
            ;
    }
}
```

The header file `finder-6m.h` contains all the needed definitions, including
Modbus parameters and registers offsets; notice how the example uses `8-N-1`
serial configuration. After saving the settings, the sketch gives us 30s to set
our 6M to use custom parameters by setting both DIP switches on the device
`DOWN`: relevant output will be displayed on the serial console.

All the configuration values we write on the Finder 6M are stored on 16-bits
registers, so we use the following function to write to them:

```cpp
boolean modbus6MWrite16(uint8_t address, uint16_t reg, uint16_t toWrite)
{
    uint8_t attempts = 3;
    while (attempts > 0)
    {
        if (ModbusRTUClient.holdingRegisterWrite(address, reg, toWrite) == 1)
        {
            return true;
        }
        else
        {
            attempts -= 1;
            delay(10);
        }
    }
    return false;
}
```

#### Reading from the Finder 6M

In the `loop()` function we are going to read the following measurements from
the newly configured Finder 6M, and print them to the serial console:

* Frequency (Hz/100).
* Active Power (W/100).
* Apparent Power (VA/100).
* Energy (kWh/100).

```cpp
void loop()
{
    int32_t frequency = modbus6MRead32(MODBUS_6M_ADDRESS, FINDER_6M_REG_FREQUENCY_100);
    int32_t activePower = modbus6MRead32(MODBUS_6M_ADDRESS, FINDER_6M_REG_ACTIVE_POWER_100);
    int32_t apparentPower = modbus6MRead32(MODBUS_6M_ADDRESS, FINDER_6M_REG_APPARENT_POWER_100);
    int32_t energy = modbus6MRead32(MODBUS_6M_ADDRESS, FINDER_6M_REG_ENERGY_100);

    Serial.println("   frequency = " + (frequency != INVALID_DATA ? String(frequency) : String("read error!")));
    Serial.println("   active power = " + (activePower != INVALID_DATA ? String(activePower) : String("read error!")));
    Serial.println("   apparent power = " + (apparentPower != INVALID_DATA ? String(apparentPower) : String("read error!")));
    Serial.println("   energy = " + (energy != INVALID_DATA ? String(energy) : String("read error!")));
}
```

All the measurements in this example are 32-bits long and they are stored in
16-bits registers using the LSW-first notation, as noted in the documentation
of the Finder 6M. This means that we need to read the two adjacent 16-bits
registers starting at the given register offset and compose the measurement,
which we do with the following function:

```cpp
uint32_t modbus6MRead32(uint8_t address, uint16_t reg)
{
    uint8_t attempts = 3;
    while (attempts > 0)
    {
        ModbusRTUClient.requestFrom(address, HOLDING_REGISTERS, reg, 2);
        uint32_t data1 = ModbusRTUClient.read();
        uint32_t data2 = ModbusRTUClient.read();
        if (data1 != INVALID_DATA && data2 != INVALID_DATA)
        {
            return data2 << 16 | data1;
        }
        else
        {
            attempts -= 1;
            delay(10);
        }
    }
    return INVALID_DATA;
}
```

## Using the Finder 6M library

To simplify all the tasks we performed in this tutorial, it is possible to use
the `Finder6M` library. In this case, the `setup()` code is a lot easier,
because the library provides built-in functions to configure both RS-485
paramaters and the Finder 6M:

```cpp
#include <Finder6M.h>

Finder6M f6m;
constexpr uint8_t MODBUS_6M_DEFAULT_ADDRESS = 1;
constexpr uint8_t MODBUS_6M_ADDRESS = 3;

void setup()
{
    Serial.begin(38400);

    if (!f6m.init())
    {
        while (1)
            ;
    }

    // Change Modbus address
    f6m.setModbusAddress(MODBUS_6M_ADDRESS, MODBUS_6M_DEFAULT_ADDRESS);
    // Set baudrate to 38400
    f6m.setBaudrate(MODBUS_6M_DEFAULT_ADDRESS, 38400);
    // Save above settings
    if (f6m.saveSettings(MODBUS_6M_DEFAULT_ADDRESS))
    {
        Serial.println("Waiting 30s while you:");
        Serial.println("1. Power OFF the 6M.");
        Serial.println("2. Set both DIP switches DOWN.");
        Serial.println("3. Power back ON the 6M.");
        delay(30000);
    }
    else
    {
        while (1)
            ;
    }
}
```

The `loop()` is also simpler, and we no longer need to write our own functions
to interact with registers:

```cpp
void loop()
{
    int32_t frequency = f6m.getFrequency100(MODBUS_6M_ADDRESS);
    int32_t activePower = f6m.getActivePower100(MODBUS_6M_ADDRESS);
    int32_t apparentPower = f6m.getApparentPower100(MODBUS_6M_ADDRESS);
    int32_t energy = f6m.getEnergy100(MODBUS_6M_ADDRESS);

    Serial.println("   frequency = " + (frequency != INVALID_DATA ? String(frequency) : String("read error!")));
    Serial.println("   active power = " + (activePower != INVALID_DATA ? String(activePower) : String("read error!")));
    Serial.println("   apparent power = " + (apparentPower != INVALID_DATA ? String(apparentPower) : String("read error!")));
    Serial.println("   energy = " + (energy != INVALID_DATA ? String(energy) : String("read error!")));
}
```

To learn more about the library head over to the [official
repository](https://github.com/dndg/Finder6M).

## Conclusion

This tutorial demonstrates how to use the `ArduinoRS485` and `ArduinoModbus`
libraries to implement the Modbus RTU protocol between the Opta and a Finder 6M
power analyzer. Additionally, it shows how it is possible to use the `Finder6M`
library to easily read measurements from a 6M.
