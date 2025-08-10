+++
date = '2025-08-10T11:01:40+02:00'
draft = false
tags = ["ble", "bluetooth", "columbus globes", "timeular", "go"]
categories = ["Programming", "Go"]
author = "Jan"
description = "Hacking the Columbus video pen and Timeular tracker"
title = "Introducing Bartolome BLE Toolkit: A Go Library for Bluetooth Low Energy Devices featuring support for Columbus Video Pens and Timeular Trackers"
+++

Working with Bluetooth Low Energy (BLE) devices can be challenging, especially when you need to manage multiple devices simultaneously with robust connection handling. That's why I created the **Bartolome BLE Toolkit** – a modular Go library. I used it to hack into the Columbus Video Pens and Timeular activity trackers, so their signal can be used to build your own applications. The library comes with extensible support for any BLE device.

## What is Bartolome BLE Toolkit?

Bartolome is a Go library that provides high-level abstractions for working with BLE devices, with support for **Columbus Video Pens** (with automatic country detection) and **Timeular activity trackers** (with side detection and multi-device support) as featured examples. The library handles the complex aspects of BLE communication – connection management, automatic reconnection, signal processing, and device-specific protocols – so you can focus on building your application logic.

The toolkit is named after Bartolomé de las Casas, as [a little jab at Columbus and a nod to the Oatmeal](https://theoatmeal.com/comics/columbus_day)

## Key Features

### Modular Architecture
The library is built with a clean, modular design where each device type is implemented as an independent package. This makes it easy to add new device types or use only the components you need.

### Robust Connection Management
One of the biggest challenges with BLE development is handling unreliable connections. Bartolome includes automatic reconnection logic, proper error handling, and connection state management that works reliably on macOS.

### Device-Specific Intelligence
Rather than just providing raw BLE access, Bartolome includes device-specific logic:

- **Columbus Video Pens**: Automatic country detection from hex signals with built-in country resolution
- **Timeular Trackers**: Side detection and tracking for productivity monitoring
- **Extensible**: Easy to add support for new device types

### Real-time Processing
Handle data from multiple devices simultaneously with clean, callback-based APIs that don't block your main application logic.

## Columbus Video Pen Integration: Country Detection Made Simple

One of Bartolome's standout features is its seamless integration with Columbus Video Pens, including automatic country detection from device signals. Here's how easy it is to get real-time country information:

```go
package main

import (
    "fmt"
    "log"
    "github.com/coded-aesthetics/bartolome-ble-toolkit/pkg/ble"
    "github.com/coded-aesthetics/bartolome-ble-toolkit/pkg/columbus"
    "github.com/coded-aesthetics/bartolome-ble-toolkit/pkg/countries"
)

func main() {
    // Create devices
    columbusDevice := columbus.NewDevice()
    manager := ble.NewManager()

    // Set up signal handler
    columbusDevice.OnSignal(func(signal []byte) error {
        countryHex, _ := columbus.SignalToCountryHex(signal)
        country, _ := countries.ResolveFromHex(countryHex)
        fmt.Printf("Country detected: %s\n", country.Name)
        return nil
    The Columbus Video Pen integration includes intelligent country detection – when the pen detects a location signal, Bartolome automatically resolves it to a human-readable country name. This makes it perfect for travel tracking, location-based applications, or geographic data collection.

### Key Columbus Features:
- **Automatic Country Resolution**: Converts hex signals to country names
- **Signal Validation**: Built-in validation for reliable data processing
- **Real-time Notifications**: Instant callbacks when new locations are detected
- **Nordic UART Protocol**: Uses standard `6e400001-b5a3-f393-e0a9-e50e24dcca9e` service)

    // Configure and connect
    deviceConfig := ble.DeviceConfig{
        Name: columbusDevice.GetName(),
        ServiceUUID: columbusDevice.GetServiceUUID(),
        CharacteristicUUID: columbusDevice.GetCharacteristicUUID(),
        NotificationHandler: columbusDevice.ProcessNotification,
    }

    if err := manager.ConnectDevices([]ble.DeviceConfig{deviceConfig}); err != nil {
        log.Fatal(err)
    }

    // Keep running...
    select {}
}
```

## Timeular Tracker Support: Multi-Device Productivity Monitoring

For productivity enthusiasts using Timeular devices, Bartolome makes it incredibly simple to monitor multiple trackers simultaneously. Here's a complete example showing how to handle both Columbus pens and multiple Timeular trackers:

```go
// Create all devices
columbusDevice := columbus.NewDevice()
timeularDevice1 := timeular.NewDeviceWithName("Timeular Tracker 1")
timeularDevice2 := timeular.NewDeviceWithName("Timeular Tracker 2")

// Set up handlers
columbusDevice.OnSignal(func(signal []byte) error {
    // Handle Columbus pen signals
    return nil
})

timeularDevice1.OnSideChange(func(deviceName string, side byte) error {
    fmt.Printf("Timeular 1 side: %d\n", side)
    return nil
})

// Configure all devices
deviceConfigs := []ble.DeviceConfig{
    {
        Name: columbusDevice.GetName(),
        ServiceUUID: columbusDevice.GetServiceUUID(),
        CharacteristicUUID: columbusDevice.GetCharacteristicUUID(),
        NotificationHandler: columbusDevice.ProcessNotification,
    },
    {
        Name: timeularDevice1.GetName(),
        ServiceUUID: timeularDevice1.GetServiceUUID(),
        CharacteristicUUID: timeularDevice1.GetCharacteristicUUID(),
        NotificationHandler: timeularDevice1.ProcessNotification,
    },
    // Add more devices as needed...
}
```

Each Timeular device can be individually configured and monitored, making it perfect for productivity dashboards, time-tracking applications, or workflow automation systems.

### Key Timeular Features:
- **Side Detection**: Automatic detection of which side (1-8) is facing up
- **Multi-Device Support**: Handle multiple trackers with individual names
- **Polling-Based Updates**: Efficient data collection without overwhelming your system
- **Custom Timeular Protocol**: Uses service `c7e70010-c847-11e6-8175-8c89a55d403c`

## Built for macOS

The library is specifically optimized for macOS development, handling the unique requirements of the macOS Bluetooth stack:

- Proper Bluetooth permission handling
- Connection management that works around macOS BLE limitations
- Scanning behavior optimized for macOS
- Robust reconnection logic tailored for macOS

## Architecture Overview

The library is organized into several key packages:

### Core BLE Management (`pkg/ble/`)
Provides the `Manager` type that handles device connections, reconnections, and state management. This is the foundation that all device-specific packages build upon.

### Device-Specific Packages
- **`pkg/columbus/`**: Columbus Video Pen integration with country detection
- **`pkg/timeular/`**: Timeular tracker support with side detection
- **`pkg/countries/`**: Country resolution from hex codes

### Examples and Documentation
The repository includes complete working examples in the `examples/` directory, from simple single-device setups to complex multi-device configurations.

## Extending the Library

Adding support for new device types is straightforward. Simply create a new package that implements the device interface:

```go
func (d *Device) GetName() string
func (d *Device) GetServiceUUID() bluetooth.UUID
func (d *Device) GetCharacteristicUUID() bluetooth.UUID
func (d *Device) ProcessNotification(deviceName string, data []byte) error
```

## Real-World Applications

Bartolome is perfect for applications that need to:

- Track user activity and location with Columbus Video Pens
- Monitor productivity and time allocation with Timeular devices
- Build IoT dashboards that aggregate data from multiple BLE sensors
- Create automation systems triggered by physical device interactions
- Develop research tools that need reliable BLE data collection

## Getting Started

Installation is simple with Go modules:

```bash
go get github.com/coded-aesthetics/bartolome-ble-toolkit
```

The repository includes comprehensive examples to get you started quickly. Check out the `examples/` directory for:

- **`columbus-only/`**: Simple Columbus Video Pen integration
- **`timeular-only/`**: Single Timeular tracker example
- **`full-setup/`**: Complete setup with all supported devices

## Why Bartolome?

Building reliable BLE applications shouldn't require wrestling with low-level connection management and device-specific protocols. Bartolome provides the abstractions you need while maintaining the flexibility to extend and customize for your specific use case.

Whether you're building a simple monitoring tool or a complex IoT system, Bartolome handles the hard parts so you can focus on what makes your application unique.

## Contributing and Future Development

The library is open source and welcomes contributions. The modular architecture makes it easy to add support for new device types, and I'm always interested in expanding the ecosystem of supported devices.

Check out the repository at [github.com/coded-aesthetics/bartolome-ble-toolkit](https://github.com/coded-aesthetics/bartolome-ble-toolkit) to get started, report issues, or contribute new features.

---

*Have questions about Bartolome or ideas for new device support? Feel free to open an issue on GitHub or reach out – I'd love to hear about what you're building!*
