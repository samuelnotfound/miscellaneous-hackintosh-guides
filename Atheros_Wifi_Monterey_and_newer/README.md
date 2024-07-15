# Restoring Atheros WiFi Functionality on macOS Monterey, Ventura, and Sonoma. 

> [!Note]
> OpenCore Legacy Patcher does not officially support being run on non-Apple Hardware. The OCLP Discord does not provide any support for hacks using OCLP. 

> Some information are based of MrLimeRunner's [sonoma-wifi-hacks](https://github.com/mrlimerunner/sonoma-wifi-hacks/blob/main/README.md) guide.

Note that macOS only has a limited support for Atheros cards.

Currently confirmed working cards:
* AR9565 (with additional patches)
* AR9287
* AR9485

### 1. Kernel

* [`corecaptureElCap.kext`](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi)
* [`IO80211ElCap.kext`](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi)
  * Only keep **AirportAtheros40.kext** from its Plugins folder.

 Set their **MinKernel** to `18.0.0` 
* [AMFIPass.kext](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Acidanthera) to partially re-enables AMFI, this can be handy if you're running into issues with an application you use related to AMFI.

Set **MinKernel** to `21.0.0`, or `20.0.0` if you're multi-booting with Big Sur.

 #### For AR9565, import the set of patches ar9565.plist from this repo under Kernel -> Patches of your config.plist
* Patches are based on ATH9Fixup source code. These patches will work with the injected **AirportAtheros40**. Wifi won't work without this.


### 2. Device Properties


| Key*   | Value      |   Type |
|--------|------------|--------|
| IOName |  | String |
| compatible|  | String |
| device-id |  | Data |

* The **IOName** allows OCLP to detect a "**Legacy Wireless**"
* **device-id** and **compatible** should be spoofed to one of the cards listed inside AirportAtheros40's Info.plist.
  
> _"...the kext has internal PCIID checks meaning simply expanding the device-id list won't work."_ - [Khronokernel](https://github.com/khronokernel/IO80211-Patches?tab=readme-ov-file#unsupported-atheros-chipsets)

Atheros cards listed inside **AirportAtheros40**'s **Info.plist**:
||`IOName` and `compatible`|`device-id`|Note|
|-|-|-|-|
|AR93xx Wireless Network Adapter| pci168c,30 | 30000000 | Used in iMac12,x |
|AR928X Wireless Network Adapter| pci168c,2a | 2A000000 | Used in iMac11,x |
|| pci106b,0086 | 00860000 ||  
|AR242x / AR542x Wireless Network Adapter | pci168c,1c | 1C000000 ||
|AR5416 Wireless Network Adapter | pci168c,23 | 23000000 ||
|AR5418 Wireless Network Adapter| pci168c,24 | 24000000 ||

Example:
* AR9287 with an IOName `pci168c,2e`, must set its `IOName` and `compatible` to `pci168c,2a`, and its `device-id` to `2A000000`.
* AR9485 with an IOName `pci168c,32`, must set its `IOName` and `compatible` to `pci168c,30`, and its `device-id` to `30000000`.

### 3. Misc 

- Set Secure Boot Model to `Disabled`.
Changing the secure boot status requires an NVRAM reset, or variables retained can cause issues with IMG4 verification in macOS. - as per [Khronokernel](https://github.com/mrlimerunner/sonoma-wifi-hacks?tab=readme-ov-file#pre-root-patching)
	- [ApECID](https://dortania.github.io/OpenCore-Post-Install/universal/security/applesecureboot.html#apecid) *cannot* be used with root patches, it needs to be disabled and remain disabled.
 
### 4. NVRAM
- Set SIP (System Integrity Protection) to a reduced state.

| Key*   | Value      |   Type |
|--------|------------|--------|
| csr-active-config | 03080000 | Data | 

 - Disable AMFI by adding the following to your boot-args.

| Key*   | Value      |   Type |
|--------|------------|--------|
| boot-args | amfi=0x80 | String |

If you run into issues with Electron based apps after disabling SIP, ie: *Discord*, *Google Chrome*, *VS Code*, add the following to your boot-arg `ipc_control_port_options=0`.

Once the changes have been applied, reboot, reset your NVRAM, and OpenCore Legacy Patcher should now show the option to apply root patches.

# Supplemental Guide: Assigning an ACPI Name

### Issue Overview
If you have already followed the guide above, but OCLP does not show a "**Legacy Wireless**". Your WiFi card is probably hidden under a PCI bridge, or not enumerated in ACPI at all. OpenCore can only overwrite properties for named devices in ACPI.

#### Example:
In this case, it ends with `pci168c,36`, which shows us it's **unnamed**, the `IOName` we try to inject is **not** applied.
![](screenshots/hackintool_pcie_tab.png)

If it has a name such as `ARPT`, The `IOName` we try to inject is applied.
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

# Other Important Notes: 
- Once your root volume has been patched, SIP must remain at least partially disabled, or you will not be able to properly boot your system.
- Delta updates are unavailable to root-patched systems. Updates will show as the full 12GB+ installers. If necessary, you can revert your patches and update, then re-apply them, but do so at your own risk.
* Alternatively, you could use chunnann's patched <a href="https://www.insanelymac.com/forum/topic/312045-atheros-wireless-driver-os-x-101112-for-unsupported-cards/?do=findComment&comment=2509900">AirPortAtheros40.kext </a> 10.11.x (El Capitan). If using it, make sure to:
  * Delete <code>CodeSignature</code> and <code>Version.plist</code>
  * Open <code>Info.plist</code>, find <code>com.apple.iokit.IO80211Family</code>, and replace it with <code>com.apple.iokit.IO80211ElCap</code>

Credits:
* MrLimeRunner's [sonoma-wifi-hacks](https://github.com/mrlimerunner/sonoma-wifi-hacks/blob/main/README.md) guide.
* [PG7](https://www.insanelymac.com/forum/topic/359007-wifi-atheros-monterey-ventura-sonoma-work/) for the idea
* [Chunnan](https://www.insanelymac.com/forum/topic/312045-atheros-wireless-driver-os-x-101112-for-unsupported-cards/?do=findComment&comment=2509900) patches from ATH9Fixup
* [Dortania](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi) for patched IO80211ElCap.kext
* [Alejandro](https://github.com/aleelmaitro/Hackintosh-Atheros-Wi-Fi-Legacy-Cards) information in regards of which `device-id` is appropriate for specific Atheros Wireless Card.
