# Controller board

The controller PCB and printhead adapter board are maintained in a separate
KiCad project. The adapter connects the controller to the printhead's flex
connector.

!!! info "Maintained separately"
    The KiCad project is not part of the Paintress software repository.
    The software repository contains the firmware. The pinout it
    expects is in `paintress-firmware/config/board_config.h`.

## Origin and methodology

The board was developed through independent reverse engineering using
oscilloscope measurements, signal analysis and public references such as
textbooks, application notes and descriptions of generic components. No
datasheets, proprietary documentation or firmware were used. The project is not
affiliated with or endorsed by a printer manufacturer.

!!! danger "High voltage"
    The hardware runs at up to **42 V DC** into reactive loads
    (piezoelectric elements) and has not been certified to any safety
    standard. Incorrect assembly, modification or operation can cause
    electric shock, component damage, fire or damage to connected equipment.
    Read [Safety](../safety.md) before building or operating the hardware.

## What the firmware expects of it

The board uses an RP2350-class MCU and 8 MB of external PSRAM on XIP
chip-select 1. PSRAM is required for swath buffering; if initialisation fails,
the firmware halts and flashes the LED rapidly. The head connections include
the PIO data bus, window selector, DAC, device enable and DAC standby/reset
lines. The board also receives the start trigger.

The DAC standby and reset lines need hardware pull-downs. The firmware drives
them LOW at the start of `main()`, but only the board can keep them low
between power-on and that first instruction.

## Wiring

The RP2350 GPIO assignments for the printhead interface (data bus, window
selector, DAC, enable, start trigger) are defined in
`paintress-firmware/config/board_config.h`.
