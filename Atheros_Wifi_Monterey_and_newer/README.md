# Restore Wi-Fi for Atheros on macOS Monterey to Sequoia

Apple dropped the kext/support for Atheros cards in macOS Mojave, and is necessary to inject older versions of kexts. Starting from Monterey, additional patches are needed to be installed using OpenCore Legacy Patcher (OCLP).

Supported chipsets that had support dropped in Mojave, these include:

- AR242x
- AR542x
- AR5416
- AR5418
- AR9280 - AR5BHB92
- AR9285 - AR5B95
- AR9287 - AR5B97
- AR9380 - AR5BXB112

Supported but required spoofing:

- AR946x (AR9462 & AR9463)
- AR9485
- AR9565

For certain AR9285/7 and AR9280 chipsets, they may report different ID that the kexts won't recognize so you will also need to apply spoof.

### 1. Kernel Extensions:
Add the following kexts to your `EFI/OC/Kexts` and your `config.plist`:
* [**AMFIPass.kext**](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Acidanthera)
* [**corecaptureElCap.kext**](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi)
* [**IO80211ElCap.kext**](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi) 
  * Right-click, and open. Navigate to the Plugins folder, and you’ll find three kexts in the folder. Delete the other two kexts and keep only **AirportAtheros40.kext**.

- Set **MinKernel**: `20.0.0` for AMFIPass
- Set **MinKernel**: `18.0.0` for IO80211 and AirportAtheros40

### 2. Device Properties

**This part is only needed for devices that needs spoofing.** These are the cards supported by AirportAtheros40, choose one from the following to spoof. 

||`IOName` and `compatible`|`device-id`|Note|
|-|-|-|-|
|AR93xx| pci168c,30 | 30000000 | Used in iMac12,x |
|AR928X| pci168c,2a | 2A000000 | Used in iMac11,x |
|| pci106b,0086 | 00860000 | Unknown |  
|AR242x / AR542x| pci168c,1c | 1C000000 ||
|AR5416| pci168c,23 | 23000000 ||
|AR5418| pci168c,24 | 24000000 ||

1. Identify your Wi-Fi card's device path.
   - Get the correct PCI device path for your Atheros WiFi card. You can do this with Hackintool. Just find the entry for the Wireless Network Controler, right-click and select "Copy Device Path"
3. Add the following properties:

| Key*   | Value      |   Type | Description |
|--------|------------|--------|--------|
| IOName |  | String | Helps OpenCore Patcher detect, and enable **"Legacy Wireless"** option. |
| device-id |  | Data | For cards that needs spoofing <i>" "...due to AirPortAtheros40 having internal PCI ID checks meaning simply expanding the device-id list won't work."</i> [source](https://github.com/khronokernel/IO80211-Patches?tab=readme-ov-file#unsupported-atheros-chipsets) |
| compatible | | String | Additional spoof|

Example:
* AR9287 has an IOName of `pci168c,2e`, can set its `IOName` and `compatible` to `pci168c,2a`, and its `device-id` to `2A000000`. 
* AR9485 with an IOName `pci168c,32`, can set its `IOName` and `compatible` to `pci168c,30`, and its `device-id` to `30000000`.
- Choose the closest one.

It should look something like this:<br>
```
DeviceProperties
└── Add
‎‎‎‎‎‎‎‎‎‎‎‎‎‎‎‎ㅤ└── PciRoot(0x0)/Pci(0x1c,0x3)/Pci(0x0,0x0)
ㅤㅤ├── device-id → Data → 2A000000
ㅤㅤ├── compatible → String → pci168c,2a
ㅤㅤ└── IOName → String → pci168c,2a
```

### 3. Misc 

- Set Secure Boot Model to `Disabled`.
     - Changing the secure boot status requires an NVRAM reset, or variables retained can cause issues with IMG4 verification in macOS. ([source](https://github.com/mrlimerunner/sonoma-wifi-hacks?tab=readme-ov-file#pre-root-patching))

### 4. NVRAM

Add the following NVRAM parameters under `Add` and `Delete`:

| Key*   | Value      |   Type |
|--------|------------|--------|
| boot-args | amfi=0x80 ipc_control_port_options=0 | String |
| csr-active-config | 03080000 | Data | 

- `ipc_control_port_options=0`: workaround if you run into issues with Electron based apps after disabling SIP, ie: *Discord*, *Google Chrome*, *VS Code*.
- `amfi=0x80`: disables AMFI, required in order for OCLP to appply root patches.
-  `csr-active-config` of `03080000` sets SIP (System Integrity Protection) to a reduced state.

Once the changes have been applied, reboot, reset your NVRAM. OpenCore Legacy Patcher should now show the option to apply root patches.

# PLEASE USE THE RELEASE VERSION OF OCLP 

# Troubleshoot
* Cannot connect to Wi-Fi
	* To work around this, manually connect using the "Other" option in the Wi-Fi menu bar or manually add the network in the "Network" preference pane.


# Supplemental Guide: Assigning an ACPI Name

### Issue Overview
If you have already followed the guide above, but OCLP does not prompt a "**Legacy Wireless**", then your Wifi card is not enumerated in ACPI at all. OpenCore can only overwrite properties for named devices in ACPI.
If OCLP does show "**Available patches: Legacy Wireless**", that means OpenCore successfully loaded the device properties we injected and you don't need to follow this guide.

#### Example:
In this case, it ends with `pci168c,36`, which tells us it's **unnamed** (not a four letter name), the `IOName` we try to inject/spoof is **not** applied.
![](screenshots/hackintool_pcie_tab.png)

If it has a name such as `ARPT` (or any four letter name, such as `PXSX` etc.), the `IOName` we try to inject/spoof is applied.
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
- Use TP-Link UB400, and add `Bluetoolfixup.kext`.
- You will also need to hide the internal bluetooth by disabling the USB port in USBMap.

# Other Important Notes: 
- Once your root volume has been patched, SIP must remain at least partially disabled, or you will not be able to properly boot your system.
- Delta updates are unavailable to root-patched systems. Updates will show as the full 12GB+ installers. If necessary, you can revert your patches and update, then re-apply them, but do so at your own risk.

This option is another way to make AR9565 Wi-Fi card work:
- Import the patches from AR9565.plist to your config, this plist includes patches taken from ATH9kFixup that patches the AirportAtheros40.kext on boot.
  - You can use CorpNewt's PatchMerge tool, or OCAT, (or OpenCore Configurator - it is easy to use, but not recommended as this app may randomly corrupt your config.plist) to import them to your config.plist.
  - However, you’ll also need to add the ATH9KInjector.kext (**NOT** Ath9kFixup, which can be downloaded from BlackDragon's ATH9kFixup repository. You **MUST** also remove the device-id or compatible device properties, but  leave the IOName property since OpenCore Legacy Patcher (OCLP) requires it to detect _Legacy Wireless_.
  - If you've followed PG7's tutorials on YouTube, you might have seen him use a patched kext shared by Chunnan on InsanelyMac. That kext is patched with the same fixes from ATH9kFixup.


Credits:
* MrLimeRunner's [sonoma-wifi-hacks](https://github.com/mrlimerunner/sonoma-wifi-hacks/blob/main/README.md) guide.
* [PG7](https://www.insanelymac.com/forum/topic/359007-wifi-atheros-monterey-ventura-sonoma-work/) for the idea
* [Dortania](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi) Patched kexts, AMFIPass (by DhinakG) 
