+++
date = '2025-08-10T10:01:40+02:00'
draft = false
tags = ["ble", "bluetooth", "timeular", "go", "cosmic-unicorn", "raspberry pi pico w", "web server"]
categories = ["Programming", "Go", "MicroPython"]
author = "Jan"
description = "Roll the dice, light up the unicorn"
title = "Using Timeular Tracker to display dice rolls on the Cosmic Unicorn"
+++

![/images/timeular.gif](/images/timeular.gif)

Have you ever wanted to create a physical display that responds to real-world interactions? In this tutorial, we'll build a fascinating IoT project that combines Bluetooth Low Energy (BLE), WiFi, and LED matrix displays to create a responsive system where rotating a physical device instantly updates a display. You need a timeular tracker and a cosmic unicorn 32x32 matrix to actually be able to build this. But even if you do not own any of these components you can still learn how to setup a web server on a Raspberry PI Pico w and how to use the bartolome-ble-toolkit to connect to bluetooth periphals.

## What We're Building

Our project creates a seamless connection between three components:
1. **Timeular Tracker** - A physical octagonal device that detects which side is facing up
2. **Go Application** - A desktop/server application that reads the tracker via BLE using the Bartolome BLE Toolkit
3. **Raspberry Pi Pico W** - A microcontroller running a web server that controls the LED display
4. **Cosmic Unicorn 32x32 LED Matrix** - A vibrant display that shows large digits

When you rotate the Timeular tracker to show side 5, the number "5" instantly appears on the LED matrix. It's like having a physical remote control for your display!

## Where is the code?

Here: [https://github.com/coded-aesthetics/timeular-cosmic-unicorn](https://github.com/coded-aesthetics/timeular-cosmic-unicorn)

## The Architecture

```
[Timeular Tracker] --BLE--> [Go App] --HTTP--> [Pico W Server] --GPIO--> [LED Matrix]
     (Physical)          (Computer)         (WiFi/Web)           (Display)
```

The data flow is beautifully simple:
1. Timeular tracker broadcasts its current side via BLE
2. Go application captures this data using Bluetooth
3. Go app immediately sends HTTP GET request to Pico W
4. Pico W receives the request and displays the digit on the LED matrix

## Hardware Requirements

- **Timeular Tracker** - The octagonal time-tracking device
- **Raspberry Pi Pico W** - Microcontroller with WiFi capability
- **Pimoroni Cosmic Unicorn** - 32x32 RGB LED matrix display
- **Computer with Bluetooth** - To run the Go application (Windows, macOS, or Linux)

## Setting Up the Cosmic Unicorn LED Display

First, let's set up our LED matrix display server on the Pico W. This creates a web server that accepts HTTP requests to display digits.

### Installing MicroPython on Pico W

1. Download the latest MicroPython UF2 file for Pico W from [micropython.org](https://micropython.org/download/rp2-pico-w/)
2. Hold the BOOTSEL button while connecting your Pico W via USB
3. Copy the UF2 file to the RPI-RP2 drive that appears
4. Install the Pimoroni MicroPython libraries for Cosmic Unicorn

### The LED Display Server Code

The Pico W runs a complete web server that can display any digit (0-9) in large, readable format across the entire 32x32 matrix:

```python
from cosmic import CosmicUnicorn
from picographics import PicoGraphics, DISPLAY_COSMIC_UNICORN
import network
import socket

# Initialize the display
cu = CosmicUnicorn()
graphics = PicoGraphics(display=DISPLAY_COSMIC_UNICORN)
cu.set_brightness(0.5)

# Define colors
WHITE = graphics.create_pen(255, 255, 255)
RED = graphics.create_pen(255, 0, 0)
GREEN = graphics.create_pen(0, 255, 0)
BLACK = graphics.create_pen(0, 0, 0)

def draw_thick_digit(digit, color=WHITE, thickness=3):
    """Draw a large digit filling most of the 32x32 display"""
    graphics.set_pen(BLACK)
    graphics.clear()
    graphics.set_pen(color)

    # Coordinates for well-proportioned digits
    left, right = 6, 26
    top, bottom = 2, 30
    middle = 14
    width = right - left
    height = bottom - top

    # Draw different digit patterns
    if digit == 1:
        x = 16 - thickness // 2
        graphics.rectangle(x, top, thickness, height)
        graphics.rectangle(x - 3, top + 2, 3, thickness)
    elif digit == 2:
        graphics.rectangle(left, top, width, thickness)
        graphics.rectangle(right - thickness, top, thickness, middle - top)
        graphics.rectangle(left, middle - thickness//2, width, thickness)
        graphics.rectangle(left, middle, thickness, bottom - middle)
        graphics.rectangle(left, bottom - thickness, width, thickness)
    # ... (additional digit patterns)

    cu.update(graphics)
```

### Web Server with Query Parameter Handling

The server accepts HTTP requests with parameters to control what's displayed:

```python
def web_server():
    # Connect to WiFi
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect('YOUR_WIFI_NAME', 'YOUR_PASSWORD')

    while not wlan.isconnected():
        time.sleep(1)

    ip = wlan.ifconfig()[0]
    print(f'Server running on http://{ip}')

    # Create web server
    addr = socket.getaddrinfo('0.0.0.0', 80)[0][-1]
    s = socket.socket()
    s.bind(addr)
    s.listen(1)

    while True:
        try:
            cl, addr = s.accept()
            request = cl.recv(1024).decode('utf-8')

            # Parse query parameters
            params = parse_query_params(request)

            if 'num' in params:
                digit = int(params['num'])
                if 0 <= digit <= 9:
                    # Choose color based on parameter
                    color = WHITE
                    if 'color' in params:
                        if params['color'] == 'red':
                            color = RED
                        elif params['color'] == 'green':
                            color = GREEN

                    draw_thick_digit(digit, color, 3)

            # Send response
            cl.send('HTTP/1.1 200 OK\r\n\r\nOK')
            cl.close()

        except Exception as e:
            cl.close()
            print(f'Error: {e}')
```

The server responds to URLs like:
- `http://192.168.1.100/?num=5` - Display white "5"
- `http://192.168.1.100/?num=3&color=red` - Display red "3"

## Setting Up the Bartolome BLE Toolkit

The Bartolome BLE Toolkit provides a clean, Go-based interface for working with Bluetooth Low Energy devices, specifically designed for Timeular trackers.

### Installing the Toolkit

```bash
go mod init timeular-display
go get github.com/coded-aesthetics/bartolome-ble-toolkit
```

### Why Bartolome BLE Toolkit?

Traditional BLE programming involves complex connection management, device discovery, and protocol handling. Bartolome abstracts all this complexity:

- **Automatic Connection Management** - Handles reconnections when devices disconnect
- **Device-Specific APIs** - Pre-built support for Timeular protocol
- **Robust Error Handling** - Graceful handling of Bluetooth stack issues

## The Go Bridge Application

Our Go application serves as the intelligent bridge between the BLE world and the HTTP world:

```go
package main

import (
    "fmt"
    "net/http"
    "time"
    "github.com/coded-aesthetics/bartolome-ble-toolkit/pkg/ble"
    "github.com/coded-aesthetics/bartolome-ble-toolkit/pkg/timeular"
)

func main() {
    // Create Timeular device with custom polling
    timeularDevice := timeular.NewDeviceWithConfig(timeular.Config{
        Name:         "Timeular Tra", // Your device name
        PollInterval: 500 * time.Millisecond,
    })

    // Create BLE manager
    manager := ble.NewManager()

    // Handle side changes
    timeularDevice.OnSideChange(func(deviceName string, side byte) error {
        // Validate the detected side
        if !timeular.IsValidSide(side) {
            fmt.Printf("⚠️  Invalid side: %d\n", side)
            return fmt.Errorf("invalid side: %d", side)
        }

        // Send HTTP request to Pico W
        picoURL := fmt.Sprintf("http://192.168.0.185/?num=%d", side)
        response, err := http.Get(picoURL)
        if err != nil {
            fmt.Printf("❌ Error updating display: %v\n", err)
            return err
        }
        defer response.Body.Close()

        fmt.Printf("🎲 Side %d -> LED Display updated\n", side)
        return nil
    })

    // Handle disconnections gracefully
    manager.SetDisconnectHandler(func(deviceName, address string, err error) {
        fmt.Printf("⚠️  Device disconnected: %v\n", err)
        fmt.Println("🔄 Attempting to reconnect...")
        timeularDevice.Reset()
    })

    // Configure and connect
    deviceConfig := ble.DeviceConfig{
        Name:               timeularDevice.GetName(),
        ServiceUUID:        timeularDevice.GetServiceUUID(),
        CharacteristicUUID: timeularDevice.GetCharacteristicUUID(),
        NotificationHandler: timeularDevice.ProcessNotification,
    }

    if err := manager.ConnectDevices([]ble.DeviceConfig{deviceConfig}); err != nil {
        log.Fatalf("❌ Connection failed: %v", err)
    }

    fmt.Println("✅ Connected! Rotate your Timeular to see changes on the LED display")

    // Keep running until interrupted
    select {}
}
```

## The Magic in Action

Here's what happens when you rotate your Timeular tracker:

1. **Physical Rotation** - You flip the Timeular from side 3 to side 7
2. **BLE Detection** - The tracker's internal accelerometer detects the new orientation
3. **Bluetooth Broadcast** - Timeular sends BLE notification with new side data
4. **Go Processing** - Bartolome toolkit receives and validates the side change
5. **HTTP Request** - Go app sends `GET http://192.168.0.185/?num=7`
6. **Display Update** - Pico W parses the request and draws large "7" on LED matrix
7. **Visual Feedback** - Bright, colorful "7" appears instantly on the 32x32 display

The entire process happens in under 100 milliseconds, creating a satisfying, real-time connection between physical and digital worlds.

## Advanced Features and Customization

### Color-Coded Sides

You can add color coding to make different sides more meaningful:

```go
timeularDevice.OnSideChange(func(deviceName string, side byte) error {
    var color string

    switch side {
    case 1, 2, 3:
        color = "green"  // Work tasks
    case 4, 5, 6:
        color = "red"    // Break time
    case 7, 8:
        color = "white"  // Other activities
    }

    url := fmt.Sprintf("http://192.168.0.185/?num=%d&color=%s", side, color)
    // ... send request
})
```

### Multiple Display Support

The architecture easily scales to control multiple displays:

```go
displays := []string{
    "http://192.168.0.185", // Kitchen display
    "http://192.168.0.186", // Office display
    "http://192.168.0.187", // Living room display
}

for _, display := range displays {
    go func(url string) {
        http.Get(fmt.Sprintf("%s/?num=%d", url, side))
    }(display)
}
```

### Time-Based Automation

Add scheduling logic to your bridge application:

```go
timeularDevice.OnSideChange(func(deviceName string, side byte) error {
    now := time.Now()

    // Only update display during work hours
    if now.Hour() >= 9 && now.Hour() <= 17 {
        // Update display
    } else {
        // Maybe dim the display or show a clock
        http.Get("http://192.168.0.185/?mode=clock")
    }

    return nil
})
```

## Troubleshooting Common Issues

### Bluetooth Connection Problems

- **Permission Issues**: Ensure your Go application has Bluetooth permissions
- **Device Discovery**: Make sure the Timeular is powered on and not connected to other apps
- **Range Issues**: Keep the Timeular within 10 meters of your computer

### WiFi and HTTP Issues

- **Network Connectivity**: Verify Pico W and computer are on the same network
- **Firewall**: Check that port 80 is accessible on the Pico W
- **IP Address Changes**: Consider using mDNS or static IP for the Pico W

### Display Problems

- **Power Supply**: Ensure adequate power for the LED matrix
- **Library Issues**: Verify Pimoroni libraries are correctly installed
- **Memory**: Monitor Pico W memory usage with complex displays

## Extending the Project

This foundation opens up countless possibilities:

### Integration with Productivity Tools

Connect your Timeular sides to:
- **Toggl/Harvest** - Automatic time tracking
- **Slack Status** - Update your status based on current activity
- **Philips Hue** - Change room lighting based on activity type
