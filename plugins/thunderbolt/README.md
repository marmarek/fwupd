# Thunderbolt™

## Introduction

Thunderbolt™ is the brand name of a hardware interface developed by Intel that
allows the connection of external peripherals to a computer.
Versions 1 and 2 use the same connector as Mini DisplayPort (MDP), whereas
version 3 uses USB Type-C.

## Firmware Format

The daemon will decompress the cabinet archive and extract a firmware blob in
an unspecified binary file format, with vendor specific header.

This plugin supports the following protocol ID:

* com.intel.thunderbolt

## GUID Generation

These devices use a custom GUID generation scheme.
When the device is in "safe mode" the GUID is hardcoded using:

* `TBT-safemode`

... and when in runtime mode the GUID is:

* `TBT-$(vid)$(pid)-native` when native, and `TBT-$(vid)$(pid)` otherwise.

Additionally for host system thunderbolt controllers another GUID is added
containing the domain designation of the controller.  This is intended to be
used for systems with multiple host controllers to disambiguate between controllers.

* `TBT-$(vid)$(pid)-native-controller$(num)`

For retimers the only GUID created is as follows:

* `TBT-$(vid)$(pid)-retimer$index`

The retimer index is oriented around the physical connection within
the machine.  It is important as multiple controllers may otherwise
identify identically.

## Update Behavior

For most devices the firmware is written to the device at runtime and the
update is applied immediately. Once complete the controller may reboot
which may cause all connected USB and TBT devices to be reenumerated.

For some devices and circumstances (such as the Dell WD19 with a new enough
kernel) the device will remain functional for the duration of the update.
The update will complete on unplug.

If a user sets `DelayedActivation` configuration option then the update will
be staged but not completed until `activate` is separately called such as at
logout or shutdown.

## Vendor ID Security

The vendor ID is set from the udev vendor, for example set to `TBT:0x$(vid)`

## Runtime Power Management

Thunderbolt controllers are slightly unusual in that they power down completely
when no thunderbolt devices are detected. This poses a problem for fwupd as
it can't coldplug devices to see if there are firmware updates available, and
also can't ensure the controller stays awake during a firmware upgrade.

On Dell hardware the `Thunderbolt::CanForcePower` metadata value is set as the
system can force the thunderbolt controller on during coldplug or during the
firmware update process. This is typically done calling a SMI or ACPI method
which asserts the GPIO for the duration of the request.

On non-Dell hardware you will have to insert a Thunderbolt device (e.g. a dock)
into the laptop to be able to update the controller itself.

## Safe Mode

Thunderbolt hardware is also slightly unusual in that it goes into "safe mode"
whenever it encounters a critical firmware error, for instance if an update
failed to be completed. In this safe mode you cannot query the controller vendor
or model and therefore the thunderbolt plugin cannot add the correct GUID used
to match it to the correct firmware.

In this case the metadata value `Thunderbolt::IsSafeMode` is set which would
allow a different plugin to add the correct GUID based on some out-of-band
device discovery. At the moment this only happens on Dell hardware.

## GUID generation for LVFS

The GUID for the controller, which must appear in the metadata when uploading an
NVM to LVFS, can be generated by a tool like `appstream-util` (with
`generate-guid` command) or by Python (with
`uuid.uuid5(uuid.NAMESPACE_DNS, 'string')`).

The format of the string used as input is "TBT-vvvvdddd", where vvvvv is the
vendor ID and dddd is the device ID, both in hex, as appear in the controller's
DROM and exposed in the relevant sysfs attributes.

If the controller is in native enumeration mode, the string "-native" is added
at the end so the format is "TBT-vvvvdddd-native".

## External Interface Access

This plugin requires read/write access to `/sys/bus/thunderbolt`.
