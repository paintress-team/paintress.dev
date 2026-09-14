# Safety

!!! danger "Read this before building or operating any hardware"
    Paintress is **experimental** and has **not** been certified to any
    safety standard (UL, CE, FCC, ANATEL or otherwise). It is **not**
    intended for commercial, medical, automotive, aerospace or any other
    safety-critical use.

## Status

Paintress is not ready for use. The code is under active development, needs
cleanup and organisation, and may change without notice.

The hardware start trigger and a full firing print have been validated on
hardware. Other checks remain incomplete: `TRIGGER_IN_PIN` has not been
verified against the board routing, and the printhead thermistor curve is
unknown. Thermal protection is not reliable yet.

## Electrical hazards

The hardware operates at up to **42 V DC** and drives piezoelectric elements,
which are reactive loads. Incorrect assembly, modification or operation can
cause electric shock, component damage, fire and permanent injury. Connected
equipment, including printheads and power supplies, can also be damaged.

The thermal monitor is observe-only by default (`SAFETY_ENFORCE = 0`). The
integrated thermistor has no available datasheet and has not been reverse
engineered, so its curve is unknown. Its conversion parameters and temperature
limits are
placeholders. The monitor logs readings but does not shut down the DAC.

**Assume there is no thermal protection.** Before enabling enforcement,
characterise the sensor and check its readings against a trusted measurement.

## Disclaimer

This project is provided for **educational and research purposes only**.

Using this design means accepting the risks that come with it. The authors
and contributors make no warranty of any kind, express or implied; accept no
liability for damage to people, property or equipment arising from use,
misuse or inability to use the design; and take on no obligation to support
it.

Checking that the design meets the safety and regulatory requirements
wherever you are is your responsibility, and it comes before building,
operating or passing it on. By using, building or distributing the project
you accept responsibility for what follows.
