# Research Documentation

This repository contains reverse engineering research and technical documentation for the Snapmaker U1 3D printer. The documentation is derived from analyzing the printer's firmware, configuration files, and Klipper source code.

The goal is to understand the hardware architecture, pin assignments, and system behavior to enable further customization and feature development for the Snapmaker U1 ecosystem.

## Documentation

### Hardware Documentation

- [**Printer Pin Documentation**](docs/PRINTER_PIN_DOCUMENTATION.md) - Comprehensive documentation of all pins, MCUs, their functions and behaviors across the 6-microcontroller system

  - Main MCU (motion control, bed, chamber)
  - Host MCU (Linux GPIO, NFC/RFID readers)
  - 4× Extruder MCUs (independent hotend control)
  - Communication protocols
  - Behavioral characteristics
  - Firmware configuration

## Related Projects

- [paxx12/SnapmakerU1-Extended-Firmware](https://github.com/paxx12/SnapmakerU1-Extended-Firmware) - Custom firmware with SSH access, USB ethernet support, and data persistence
- [paxx12/v4l2-mpp](https://github.com/paxx12/v4l2-mpp) - Hardware-accelerated camera support for Rockchip MPP/VPU

## Contributing

This documentation is generated through analysis of the printer's firmware and configuration. Contributions that improve accuracy or add additional reverse engineering findings are welcome.
