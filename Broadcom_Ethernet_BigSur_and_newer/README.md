# Broadcom Ethernet Patch Guide (macOS Big Sur – Tahoe)

Restoring functionality for legacy Broadcom Ethernet chipsets on macOS Big Sur (11) through macOS Tahoe (26).


### Chipset Support Overview

#### Natively Supported Devices
The following Broadcom chipsets are supported by kext as-is and **do not require** binary kernel patches or `device-id` spoofing:

| Model | PCI ID |
| :--- | :--- |
| **BCM5764M** | `pci14e4,1684` |
| **BCM57761** | `pci14e4,16b0` |
| **BCM57762** | `pci14e4,1682` |
| **BCM57765** | `pci14e4,16b4` |
| **BCM57766** | `pci14e4,1686` |

#### Patched & Spoofed Supported Devices
All chipsets listed below require both `device-id` spoofing to `pci14e4,16b4` and the binary kernel patch outlined in this guide:

| Model | Device ID (HEX) |
| :--- | :--- |
| **BCM5700** | `14e4:1644` |
| **BCM5701** | `14e4:1645` |
| **BCM5702** | `14e4:1646` |
| **BCM5703** | `14e4:1647` |
| **BCM5717** | `14e4:1655` / `14e4:1665` |
| **BCM5718** | `14e4:1656` |
| **BCM5719** | `14e4:1657` |
| **BCM5725** | `14e4:1643` |
| **BCM5727** | `14e4:16f3` |
| **BCM5761** | `14e4:1688` |
| **BCM5762** | `14e4:1687` |
| **BCM57760** | `14e4:1690` |
| **BCM57764** | `14e4:1642` |
| **BCM57767** | `14e4:1683` |
| **BCM57781** | `14e4:16b1` |
| **BCM57782** | `14e4:16b7` |
| **BCM57785** | `14e4:16b5` |
| **BCM57786** | `14e4:16b3` |
| **BCM57787** | `14e4:1641` |
| **BCM57788** | `14e4:1691` |
| **BCM57790** | `14e4:1694` |
| **BCM57791** | `14e4:16b2` |
| **BCM57795** | `14e4:16b6` |
| **BCM5785** | `14e4:1699` / `14e4:16a0` |
| **BCM5787M** | `14e4:1693` |
| **Generic Adapter** | `14e4:1689` |

---

### Prerequisites

Download [**CatalinaBCM5701Ethernet.kext**](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Ethernet) provided by OpenCore Legacy Patcher (OCLP).

---

### OpenCore Configuration

#### 1. Add Device Properties
In your `config.plist`, navigate to **DeviceProperties -> Add** under your Ethernet controller PCI path (e.g., `PciRoot(0x0)/Pci(0x1C,0x0)/Pci(0x0,0x0)`):

| Key | Type | Value |
| :--- | :--- | :--- |
| `device-id` | Data | `B4160000` |
| `compatible` | String | `pci14e4,16b4` |

#### 2. Kext Injection
1. Place `CatalinaBCM5701Ethernet.kext` in your `OC/Kexts/` folder.
2. Add the kext entry to **Kernel -> Add** in your `config.plist`.
3. Set `MinKernel` to `20.0.0` for this kext entry.

#### 3. Essential Kernel Patch
Add the following entry under **Kernel -> Patch**:

| Key | Value |
| :--- | :--- |
| **Identifier** | `com.apple.iokit.CatalinaBCM5701Ethernet` |
| **Comment** | `Broadcom BCM577XX Patch` |
| **Find** | `E8CA9EFF FF668983 00050000` |
| **Replace** | `B8B41600 00668983 00050000` |
| **MinKernel** | `20.0.0` |
| **Count** | `1` |
| **Enabled** | `true` |

---

### Cosmetic Model Name Fix (Optional)

To display the correct model name in System Information, add a cosmetic patch under **Kernel -> Patch**.

#### Example Patch (57765 ➔ 57785):

| Key | Value |
| :--- | :--- |
| **Identifier** | `com.apple.iokit.CatalinaBCM5701Ethernet` |
| **Comment** | `SysReport 57765 -> 57785 (Cosmetic)` |
| **Find** | `3537373635` |
| **Replace** | `3537373835` |
| **MinKernel** | `20.0.0` |
| **Count** | `0` |
| **Enabled** | `true` |

Example: 3 <kbd>5</kbd> 3 <kbd>7</kbd> 3 <kbd>7</kbd> 3 <kbd>**6**</kbd> 3 <kbd>**5**</kbd> -> 3 <kbd>5</kbd> 3 <kbd>7</kbd> 3 <kbd>7</kbd> 3 <kbd>**8**</kbd> 3 <kbd>**5**</kbd>

### Credits

* **[Sunki](https://www.applelife.ru/threads/patching-applebcm5701ethernet-kext.27866/page-8#post-930901)** & **[Acidanthera](https://github.com/acidanthera/OpenCorePkg/blob/cb591b7671215b31dc4a2bc5b1e9da9c92eaebf4/Docs/Sample.plist#L837)** — Original driver patch sources
* **[Andrey1970AppleLife](https://www.applelife.ru/threads/patching-applebcm5701ethernet-kext.27866/page-9#post-1031837)** — Patch guide reference and cosmetic patches
* **Dortania / Khronokernel** — Pre-patched `CatalinaBCM5701Ethernet.kext` 
