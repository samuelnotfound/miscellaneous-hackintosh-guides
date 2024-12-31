# Restore Wi-Fi for Atheros on macOS Monterey to Sequoia

Apple had dropped support for Atheros cards on macOS Mojave. It now requires injecting old kexts starting Mojave, and installing patches using OCLP starting Monterey.

Supported chipsets that had support dropped in Mojave, these include:

- AR242x
- AR542x
- AR5416
- AR5418
- AR9280 - AR5BHB92
- AR9285 - AR5B95
- AR9287 - AR5B97
- AR9380 - AR5BXB112

Supported but requires spoofing:

- AR946x (AR9462 & AR9463)
- AR9485
- AR9565

#### Download:

- [**AMFIPass.kext**](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Acidanthera)
  -  **MinKernel**: `20.0.0`
* [**corecaptureElCap.kext**](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi)
  -  **MinKernel**: `18.0.0`
* [**IO80211ElCap.kext**](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi)
  * Only keep **AirportAtheros40.kext** in its Plugins folder
  * **MinKernel**: `18.0.0`

#### Add Device Properties

| Key*   | Value      |   Type | Description |
|--------|------------|--------|--------|
| IOName |  | String | Helps OpenCore Patcher detect, and enable **"Legacy Wireless"** option. |
| device-id |  | Data | For cards that needs spoofing <i>" "...due to AirPortAtheros40 having internal PCI ID checks meaning simply expanding the device-id list won't work."</i> - [Khronokernel](https://github.com/khronokernel/IO80211-Patches?tab=readme-ov-file#unsupported-atheros-chipsets) |
| compatible | | String | Additional spoof|

List of devices supported by AirportAtheros40:
||`IOName` and `compatible`|`device-id`|Note|
|-|-|-|-|
|AR93xx Wireless Network Adapter| pci168c,30 | 30000000 | Used in iMac12,x |
|AR928X Wireless Network Adapter| pci168c,2a | 2A000000 | Used in iMac11,x |
|| pci106b,0086 | 00860000 ||  
|AR242x / AR542x Wireless Network Adapter | pci168c,1c | 1C000000 ||
|AR5416 Wireless Network Adapter | pci168c,23 | 23000000 ||
|AR5418 Wireless Network Adapter| pci168c,24 | 24000000 ||

Example:
* AR9287 has an IOName of `pci168c,2e`, can set its `IOName` and `compatible` to `pci168c,2a`, and its `device-id` to `2A000000`.
* AR9485 with an IOName `pci168c,32`, can set its `IOName` and `compatible` to `pci168c,30`, and its `device-id` to `30000000`.

Choose the closest one.

#### Misc 

- Set Secure Boot Model to `Disabled`.
     - Changing the secure boot status requires an NVRAM reset, or variables retained can cause issues with IMG4 verification in macOS.
       - According to [Khronokernel](https://github.com/mrlimerunner/sonoma-wifi-hacks?tab=readme-ov-file#pre-root-patching)

#### NVRAM

Add the following NVRAM parameters under `Add` and `Delete`:

| Key*   | Value      |   Type |
|--------|------------|--------|
| boot-args | amfi=0x80 ipc_control_port_options=0 | String |
| csr-active-config | 03080000 | Data | 

- `ipc_control_port_options=0`: workaround if you run into issues with Electron based apps after disabling SIP, ie: *Discord*, *Google Chrome*, *VS Code*.
- `amfi=0x80`: disables AMFI, required in order for OCLP to appply root patches.
-  `csr-active-config` of `03080000` sets SIP (System Integrity Protection) to a reduced state.

Once the changes have been applied, reboot, reset your NVRAM. OpenCore Legacy Patcher should now show the option to apply root patches.

# Troubleshoot
* Cannot connect to Wi-Fi
	* To work around this, manually connect using the "Other" option in the Wi-Fi menu bar or manually add the network in the "Network" preference pane.


# Supplemental Guide: Assigning an ACPI Name

### Issue Overview
If you have already followed the guide above, but OCLP does not prompt a "**Legacy Wireless**", then your Wifi card is not enumerated in ACPI at all. OpenCore can only overwrite properties for named devices in ACPI.

#### Example:
In this case, it ends with `pci168c,36`, which tells us it's **unnamed** (not a four letter name), the `IOName` we try to inject/spoof is **not** applied.
![](screenshots/hackintool_pcie_tab.png)

If it has a name such as `ARPT`(has a four letter name), the `IOName` we try to inject/spoof is applied.
![](screenshots/hackintool_pci1683,36_to_ARPT.png)


### Assign an ACPI Name

1. Download, and run Hackintool
2. Identify PCI Path, and Debug value
* Navigate to the PCIe tab. Identify your WiFi card and note its ACPI path and debug values. For instance:

![](screenshots/hackintool_pcie_tab.png)

**`PCI0`**<sup> @0 /</sup> **`RP04`**<sup> @1C,3 /</sup> pci168c,36<sup> @0</sup>
* Path: PCI0.RP04 - actually the ACPI path for it's parent/PCI Bridge
* Debug: 02 00 0


Download the sample SSDT, and edit it according to your values:

```asl
DefinitionBlock ("", "SSDT", 2, "ARPT", "WIFIPCI", 0x00001000)
{
    External (_SB_.PCI0.RP04, DeviceObj)  // Replace "PCI0.RP04" with your WiFi's parent/PCI Bridge ACPI path

    Scope (_SB.PCI0.RP04)  // Replace "PCI0.RP04", same as above
    {
        Device (ARPT) // We assign a name for "pci168c,36" as "ARPT", "ARPT" is the ACPI name of WiFi card in Macs.
        {
            Name (_ADR, 0x02000000) // Add your debug value here, e.g., "02 00 0" becomes 0x02000000, `00 1C 4` becomes 0x001C40000
        }
    }
}
```

Save the edited SSDT file and add it to your OpenCore ACPI folder. Ensure your config.plist is updated to include the new SSDT by adding it to the ACPI section of your config.plist.

After restart, in my case, **PCI0**<sup> @0 /</sup> **RP04**<sup> @1C,3 /</sup> `pci168c,36`<sup> @0</sup> would now be **PCI0**<sup> @0 /</sup> **RP04**<sup> @1C,3 /</sup>`ARPT`<sup> @0</sup>

Before:
![](screenshots/hackintool_pcie_tab.png)
After:
![](screenshots/hackintool_pci1683,36_to_ARPT.png)


You can now see the `IOName` is properly injected/spoofed. OCLP will now recognize the spoofed `IOName` of an iMac11,x Atheros card - of which OCLP supports.

|Before|After|
|-|-|
|![](screenshots/real_ioname.png)|![](screenshots/spoofed_ioname.png)|

Open the OCLP app, then apply root patches.

# Bluetooth (Monterey+)
Use TP-Link UB400, and add `Bluetoolfixup.kext`.

# Other Important Notes: 
- Once your root volume has been patched, SIP must remain at least partially disabled, or you will not be able to properly boot your system.
- Delta updates are unavailable to root-patched systems. Updates will show as the full 12GB+ installers. If necessary, you can revert your patches and update, then re-apply them, but do so at your own risk.


Credits:
* MrLimeRunner's [sonoma-wifi-hacks](https://github.com/mrlimerunner/sonoma-wifi-hacks/blob/main/README.md) guide.
* [PG7](https://www.insanelymac.com/forum/topic/359007-wifi-atheros-monterey-ventura-sonoma-work/) for the idea
* [Dortania](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi) Patched kexts, AMFIPass (by DhinakG) 
