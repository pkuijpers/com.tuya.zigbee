# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Homey app (SDK 3) that adds Zigbee support for Tuya-branded and white-label devices. It supports 100+ device types including sensors, plugs, switches, lights, curtain motors, thermostats, and remotes.

## Commands

```bash
npm install          # Install dependencies
npm run lint         # Run ESLint
```

To deploy/test on Homey, use the Homey CLI (`homey app run`, `homey app install`).

## Architecture

### Entry Point
- `app.js` - Main Homey.App class, registers global flow cards

### Driver Structure
Each device type has a driver in `/drivers/<device_type>/`:
- `device.js` - Device class handling Zigbee communication
- `driver.compose.json` - Device metadata, Zigbee identifiers (manufacturerName/productId), capabilities
- `driver.settings.compose.json` - Device settings schema
- `driver.flow.compose.json` - Device-specific flow cards (optional)
- `assets/` - Device images

### Shared Libraries (`/lib/`)

**Tuya Protocol Implementation:**
- `TuyaSpecificCluster.js` - Defines Tuya's proprietary Zigbee cluster (ID: 61184) with commands: datapoint, reporting, response, reportingConfiguration
- `TuyaSpecificClusterDevice.js` - Base class for Tuya devices, provides `writeBool()`, `writeData32()`, `writeString()`, `writeEnum()`, `writeRaw()` methods for sending datapoint commands
- `TuyaDataPoints.js` - Centralized datapoint definitions for all device types (thermostats, curtains, sensors, switches, etc.)
- `TuyaProtocol.md` - Protocol documentation

**Other Libraries:**
- `TuyaZigBeeLightDevice.js` - Base class for RGB/tunable light devices
- `TuyaColorControlCluster.js`, `TuyaOnOffCluster.js` - Custom cluster implementations
- Bound clusters for IAS Zone, OnOff, WindowCovering

### Homey Compose (`/.homeycompose/`)
- `app.json` - App metadata and configuration
- `capabilities/` - Custom capability definitions (thermostat_preset, soil_moisture, etc.)
- `flow/` - Global flow card definitions
- `drivers/` - Shared driver configurations

### Device Communication Patterns

**Standard Zigbee devices** (plugs, basic sensors): Extend `ZigBeeDevice` and use standard clusters (ON_OFF, etc.)

**Tuya-specific devices** (thermostats, curtain motors, complex sensors): Extend `TuyaSpecificClusterDevice` and communicate via datapoints:
```javascript
// Register the Tuya cluster
Cluster.addCluster(TuyaSpecificCluster);

class MyDevice extends TuyaSpecificClusterDevice {
    async onNodeInit({ zclNode }) {
        // Listen for reports from device
        zclNode.endpoints[1].clusters.tuya.on("reporting", value => this.processReport(value));

        // Send commands using datapoints
        await this.writeBool(DP_ID, true);
        await this.writeData32(DP_ID, 250);  // e.g., temperature * 10
        await this.writeEnum(DP_ID, 0);
    }
}
```

### Tuya Datapoint System
Tuya devices use a proprietary datapoint (DP) protocol instead of standard Zigbee clusters. Each DP has:
- ID (number identifying the function)
- Datatype: RAW (0), BOOL (1), VALUE/uint32 (2), STRING (3), ENUM (4), FAULT/bitmap (5)

Import datapoints from `TuyaDataPoints.js`:
```javascript
const { V1_THERMOSTAT_DATA_POINTS } = require('../../lib/TuyaDataPoints');
```

## Adding New Device Support

1. Create driver directory in `/drivers/<device_name>/`
2. Create `driver.compose.json` with Zigbee manufacturerName and productId arrays
3. Create `device.js` extending appropriate base class
4. For Tuya protocol devices, define datapoints in `TuyaDataPoints.js` if not already present
5. Run `homey app run` to generate `app.json` from compose files
