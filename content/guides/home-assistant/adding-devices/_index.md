---
title: Adding your first Device to Home Assistant
resources:
  - name: zha-state
    src: "ZHA_state.png"
    title: |
      The Zigbee Home Automation card in the Settings of Home Assistant
  - name: searching_for_zigbee_device
    src: "searching_for_zigbee_device.png"
    title: |
      The Zigbee Home Automation is looking for Zigbee devices in proximity in
      pairing mode.
  - name: aqara_motion_sensor_found
    src: "aqara_motion_sensor_found.png"
    title: |
      The Aqara Motion Sensor will show up as a card outlined in green once it
      has been discovered and added to Home Assistant.
  - name: create_automation_from_motion_sensor
    src: create_automation_from_motion_sensor.png
    title: |
      Template automations available for motion sensors.
  - name: aqara_motion_sensor_details
    src: aqara_motion_sensor_details.png
    title: |
      Properties of the Aqara Motion Sensor in the ZigBee Home Automation.
---

For the purpose of this tutorial we are using the following setup:

1. A Linux based installation of Home Assistant either using a container or the
   [Home Assistant Operating
   System](https://developers.home-assistant.io/docs/operating-system/)
2. A ZigBee connector compatible with the [ZigBee Home Automation
   Integration](https://www.home-assistant.io/integrations/zha), e.g. the
   [Nortek GoControl USB
   Zigbee/Zwave](https://www.nortekcontrol.com/products/2gig/husbzb-1-gocontrol-quickstick-combo/)
   or the [Sonoff Zigbee 3.0 USB Dongle
   plus](https://sonoff.tech/product/diy-smart-switch/sonoff-zigbee-3-0-usb-dongle-plus-e/)
3. [Aqara Motion Sensor](https://www.aqara.com/en/human_motion_sensor.html) or
   [Aqara Motion Sensor P1](https://www.aqara.com/en/product/motion-sensor-p1)
   (both use Zigbee)

At this point you should have the ZigBee Home Automation set up and ready to
talk to the Zigbee network. It should look something like this under
'Settings' → 'Devices' (it might have no devices if you have not added any):

{{< img name="zha-state" size="origin" >}}


Adding a ZigBee device is very similar to pairing a Bluetooth device on your
phone. The device needs to be set into "pairing" mode and then the server (in
this case Home Assistant) needs to search for it.

First, you need to set the device into pairing mode. For the Aqara Motion
sensor, press the small, indented button for five seconds.

![aquara_motion_sensor](https://manuals.plus/wp-content/uploads/2022/03/Aqara-MS-S02-Motion-Sensor-P1-FIG-1.png)


Once it's in pairing mode, start the device search in Home Assistant. The device
search can be found under Settings → Devices and Services → Zigbee (click on the
devices link) → Add device.

{{< img name="searching_for_zigbee_device" size="origin" >}}

It might take anywhere from 10-30 seconds for the device to show up. Sometimes
it is necessary to "wake" the device. Just press the indented button which
should make the blue LED flash.

Once the device has been found Home Assistant will display the following:

{{< img name="aqara_motion_sensor_found" size="origin" >}}

The icons show that it found a battery, temperature, motion and illumination
sensors on the device.

After picking an 'area' the device is ready to be used (it might take another
30-60 seconds for the search to finish even after it has found the device, but
you can close this page beforehand). For the purpose of this guide, the motion
sensor has been assigned to the 'Library' area:

{{< img name="aqara_motion_sensor_details" size="origin" >}}

The battery isn't always correct for some reason. But the temperature,
illuminance and all other sensors should work correctly.

The last step is choosing what to automate. On the sensor screen, click on the
'+' next to automations. The following options will show up

{{< img name="create_automation_from_motion_sensor" size="origin" >}}

A simple choice is to click on 'started detecting motion'. This will take you to
the 'New Automation' section.
